---
title: CloudWatch Dashboards
summary: Practical reference for CloudWatch's automatic and custom dashboards — widgets, JSON structure, cross-account/cross-Region graphs, variables, sharing, and pricing
last_verified: 2026-09-27
categories: [Amazon Web Services, Cloud monitoring, Observability]
---

# CloudWatch Dashboards

This article is the practical companion to [Amazon CloudWatch](amazon-cloudwatch.md): what actually shows up on the CloudWatch home page without any setup (**automatic dashboards**), and how to build, share, and parameterize your own (**custom dashboards**).

## Automatic dashboards

The CloudWatch console home page is itself a dashboard, generated with zero configuration from every AWS service used in the account:

- **Alarms by AWS service** — a per-service breakdown of alarm state, plus a short list of alarms currently in `ALARM` or that most recently changed state.
- **Cross-service dashboard** — metric graphs pulled automatically from every AWS service in use. This view can be narrowed to a single AWS service, or to a specific resource group.
- **Default dashboard (`CloudWatch-Default`)** — if you create and name a custom dashboard exactly `CloudWatch-Default`, it renders inline on the home page below the automatic sections. This is the intended way to pin your own application metrics into the automatic view rather than only seeing AWS-service metrics.

Automatic dashboards are **always free**, regardless of how much data they display, and only ever show data from the account you're currently signed into — even from a cross-account-observability monitoring account, the home page stays account-scoped (custom cross-account dashboards are a separate, deliberate thing you build; see below).

**The one real customization knob on the cross-service dashboard**: select a service at the top of the Overview page → **Actions** → clear **"Show on cross service dashboard"**. This hides that service's metrics from the cross-service view (its alarms still appear in the Alarms views) — useful for suppressing noisy or irrelevant services from the account-wide default view. Beyond this toggle and the `CloudWatch-Default` trick, the automatic dashboard's layout, sections, and per-service graphs are fixed — there's no reordering, no per-widget editing, and no way to add arbitrary widgets to it the way you can a custom dashboard.

## Custom dashboards

A custom dashboard is a named, ordered grid of widgets that you build via console drag-and-drop, the CLI, or the `PutDashboard` API. Widgets can pull from metrics, alarms, Logs Insights queries, or a Lambda-backed custom source, and — with cross-account observability set up — from other AWS accounts and Regions in the same view.

### Widget types

| Widget | What it shows |
|---|---|
| Line | Time series comparison across periods; supports mini-map zoom to inspect a section without losing the zoomed-out view. |
| Stacked area | Like line, but values stack to show composition/total. |
| Bar | Metric values as bars, useful for period-over-period comparison. |
| Pie | Proportional breakdown at a single point in time. |
| Number | Latest value(s) with a sparkline showing recent trend — good for at-a-glance KPI tiles. |
| Gauge | A value plotted within a fixed min/max range (e.g. CPU%), for quick threshold-style reading. |
| Text/image | Static Markdown — headings, bold/italic/strikethrough, links, bulleted/numbered lists, monospace code blocks, plus two CloudWatch-specific extras: pipe-delimited **tables** and clickable **buttons** (`[button:Text](url)` / `[button:primary:Text](url)`). Also supports **images**: `![alt](url)` from an externally hosted image (`.jpeg`/`.png`/`.svg`/`.gif`/`.webp`), or a fully inline **base64 data URI** (`![alt](data:image/png;base64,...)`) — the latter needs no external hosting at all. A dedicated link syntax also exists for deep-linking to *another dashboard in the same account*: `[text](#dashboards:name=other-dashboard-name)`. **Caveat**: AWS's own public Markdown reference page (docs.aws.amazon.com/awsconsolehelpdocs/latest/gsg/aws-markdown.html) does not document image support or the dashboard-deep-link syntax as of this writing — both are confirmed only by the console's own prefilled Text-widget template, which is ahead of the published docs. No client-side JS or arbitrary HTML — that's exclusive to the Custom widget below. |
| Alarm (single) | One alarm's metric graph plus its current state overlay. |
| Alarm status | A grid of up to **100 alarms**, names + state only, no graphs — a dense health-board view. |
| Table | Raw datapoints and per-statistic summary (Min/Max/Sum/Average), undoing the graph's abstraction when you need exact numbers. |
| Log (Logs Insights) | Results of a saved Logs Insights query, rendered as a table or time series. |
| Metrics Explorer | Auto-updating graphs grouped by tag or shared resource property (e.g. instance type) — tracks fleet membership as resources are created/deleted, so you don't have to manually add new instances to a graph. Its own `type: "explorer"` JSON has a distinct property set: `aggregateBy` (a key + `AVG`/`MIN`/`MAX`/`STDDEV`/`SUM`), `labels` (tag or resource-property filters, from a fixed whitelist of ~30+ `resourceType` values covering EC2/Lambda/RDS/etc.), `splitBy`, and `widgetOptions` (`legend.position`, `rowsPerPage`, `widgetsPerRow`, `stacked`, `view`). |
| Custom widget | Output of a Lambda function you point the widget at — for pulling in anything not natively supported (third-party data, custom HTML/SVG). |

Graphs can be **linked** so that zooming one zooms all linked graphs together, and any metric can be added manually to a graph even if it hasn't published data in the last 14 days (normal metric search only surfaces metrics active in that window).

### Query Studio vs. the classic Metrics console

When adding a metric widget, the console offers a "preferred experience" choice — **Metrics console** (the classic flow) or **Query Studio** — as a per-widget decision at add-time, not a one-time account setting.

**Query Studio** is a newer, **public-preview** feature (launched April 2026, and still limited to five Regions: `us-east-1`, `us-west-2`, `ap-southeast-2`, `ap-southeast-1`, `eu-west-1`). Its headline capability is bringing **native PromQL** querying into CloudWatch for the first time, unified with CloudWatch Metric Insights in one interface, letting you query both AWS-vended metrics and OpenTelemetry (OTLP) metrics without switching consoles. It offers a **Builder mode** (visual form, autocomplete) and an **Editor mode** (raw PromQL with syntax highlighting); "Add to dashboard" saves the query as a widget that keeps re-running on the dashboard's normal refresh cadence.

This is why it has noticeably fewer output options than the classic path: Query Studio is a **query-first tool scoped to PromQL/Metric Insights results**, rendering only as a graph or table — it's not a general widget builder. The classic Metrics console flow is what feeds the full widget-type catalog above (Line, Bar, Pie, Number, Gauge, Metrics Explorer, etc.). In short: pick Query Studio for PromQL/OTel-metric power, pick the classic Metrics console for the full range of visualization types.

### Multi source query: non-CloudWatch data sources

The Add Widget dialog also offers **"Create data source"**, letting a widget pull from Amazon OpenSearch Service, Amazon Managed Service for Prometheus (or self-hosted Prometheus), Amazon RDS for MySQL/PostgreSQL, Amazon S3 (CSV files), or Microsoft Azure Monitor — configured once under **Settings → Metrics data sources**, then reused across any dashboard. This is the "Multi source query" feature.

**It is not a native, code-free connector** — clicking "Create data source" deploys a **CloudFormation stack** that provisions a **Lambda function** (with its own execution IAM role; the console requires you to explicitly acknowledge that CloudFormation will create IAM resources) to act as the connector. Credentials (DB password, Azure client secret, Prometheus basic-auth token) are stored in **AWS Secrets Manager** and injected into the Lambda's environment. Private/VPC-only sources (internal RDS, self-hosted Prometheus) additionally require attaching the connector to a VPC/subnet/security group, plus a Secrets Manager VPC endpoint and, for RDS, an RDS interface VPC endpoint. In other words: you don't write the Lambda yourself, but AWS is still running one on your behalf — a managed, templated version of the same mechanism as the Custom widget above, not a fundamentally different architecture.

Query language is source-specific, not unified: Prometheus/AMP use **PromQL**; RDS uses real **SQL**, with CloudWatch injecting `$start.iso`/`$end.iso`/`$period` variables; OpenSearch and Azure Monitor use a point-and-click builder with no raw query exposed; S3 CSV needs no query at all — just a bucket/key with a required format (timestamp-first column, header row, ISO timestamps). One universal gotcha: **multi-line queries aren't supported** — line feeds collapse to a space, which can silently break a query containing a single-line comment.

**Azure Monitor's scope is narrower than "Azure data" might suggest.** After providing a tenant ID/client ID/client secret (an Entra app registration/service principal), the query builder is: subscription → resource group → resource → metric namespace → metric → aggregation, with optional dimension filtering — exactly Azure Monitor's **platform Metrics** surface (VM CPU, Storage Account transactions, SQL DTU, App Service latency, and similar resource-scoped numeric time series). **Microsoft Entra ID / Azure AD data (sign-in logs, audit logs, directory activity) is not reachable through this connector** — AWS's documentation for it makes no mention of Log Analytics, KQL, Azure AD, Entra ID, sign-in logs, or audit logs anywhere. That data lives in a structurally different Azure service, **Azure Monitor Logs** (a Log Analytics workspace, queried in KQL) or the Microsoft Graph API — neither is wired into this feature. Reaching Entra ID data from CloudWatch would require a different path entirely: a custom (Lambda-backed) widget calling the Graph API directly, or an export from Log Analytics into something this connector can reach (e.g. via S3 CSV). AWS's docs also don't state the exact Azure RBAC role required for the app registration — that's determined by whatever role you grant it on the Azure side, not prescribed by AWS; a **Monitoring Reader**-scoped role is the natural minimum for what the connector actually does (resource-level metric reads), but confirm against Microsoft's own service-principal documentation before assuming it.

**Does the result become a real CloudWatch metric?** Not literally — there's no evidence of `PutMetricData`-style ingestion into a CloudWatch namespace. But it isn't purely read-only display either: AWS's docs confirm you can build **both dashboard widgets and alarms** directly from a multi-source query result, so it behaves like a metric anywhere the console lets you pick one, while architecturally remaining a live pass-through query executed by the connector Lambda on each refresh, not a stored datapoint.

### Dashboard body structure

A dashboard is stored as a JSON document (`DashboardBody`) — a `widgets` array, each with `type`, position (`x`, `y`), size (`width`, `height`), and a `properties` object specific to that widget type. The console's **Actions → View/edit source** exposes this JSON directly, and it's the easiest way to hand-author or template dashboards.

**Dashboard-level properties** (apply to the whole dashboard, not one widget): `start`/`end` (ISO 8601, or relative like `-PT6H`/`-P3M`) set the default time range on load; `periodOverride` is `auto` (period adapts to the selected time range) or `inherit` (each graph keeps its own configured period regardless of the dashboard's range); `variables` is an array holding up to **25** dashboard variables.

**Widget-level customization** (inside a metric widget's `properties`, beyond `metrics`/`view`/`period`/`stacked` already shown below):

- `yAxis.left` / `yAxis.right` — each independently configurable with `min`, `max`, `label`, `showUnits` (default `true`).
- `annotations.horizontal` / `annotations.vertical` — threshold lines/bands: `value`, `label`, `color` (hex), `fill` (`above`/`below`/`none` for horizontal, `before`/`after`/`none` for vertical), `visible`. A widget can carry an alarm annotation *or* horizontal/vertical annotations, not both at once.
- `legend.position` — `right` | `bottom` | `hidden`.
- `timezone` — an offset string (e.g. `+0300`) for how the graph renders time.
- `liveData` — boolean; shows unaggregated data published in the last minute.
- `sparkline` — boolean; only applies to the `singleValue`/Number view.
- Per-metric overrides — each metric entry inside the `metrics` array can itself carry `color`, `label`, `period`, `stat`, `visible`, `yAxis`, overriding the widget-level defaults for just that one line.
- Table view adds its own `table` object: `layout` (`vertical`/`horizontal`), `stickySummary`, `showTimeSeriesData`, `summaryColumns` (a subset of `MIN`/`MAX`/`SUM`/`AVG`).
- `view: "gauge"` requires `yAxis.left.min`/`.max` to be set — a gauge has no default range.

Not customizable via any documented dashboard JSON property: a dark/light theme toggle and an auto-refresh interval are account/browser-level UI preferences, not part of `DashboardBody` — they aren't something you can set per-dashboard or hand off to a viewer.

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Checkout latency",
        "view": "timeSeries",
        "stacked": false,
        "period": 300,
        "metrics": [
          ["MyApp/Orders", "ProcessingTime", "Service", "checkout", { "stat": "Average" }]
        ]
      }
    },
    {
      "type": "text",
      "x": 0, "y": 6, "width": 24, "height": 2,
      "properties": { "markdown": "## On breach: check the [runbook](https://example.com/runbook)" }
    }
  ]
}
```

A single-alarm widget references the alarm by ARN rather than by metric:

```json
{
  "type": "metric",
  "x": 6, "y": 0, "width": 6, "height": 6,
  "properties": {
    "title": "over50",
    "view": "timeSeries",
    "annotations": { "alarms": ["arn:aws:cloudwatch:us-east-1:111122223333:alarm:over50"] }
  }
}
```

A Logs Insights widget uses `type: "log"` with the query text inline:

```json
{
  "type": "log",
  "x": 0, "y": 6, "width": 24, "height": 6,
  "properties": {
    "query": "SOURCE 'route53test' | fields @timestamp, @message\n| sort @timestamp desc\n| limit 20",
    "view": "table"
  }
}
```

### Cross-account, cross-Region dashboards

**Cross-Region alone needs no extra setup.** Any widget's `metrics` array can mix Regions within the same account — set `region` on the widget or on individual metric entries — with zero Observability Access Manager configuration. This is a basic capability of custom dashboards, available to every account by default.

**Cross-*account*** is the part that requires setup. With [CloudWatch cross-account observability](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Unified-Cross-Account.html) configured (a monitoring account linked to one or more source accounts via Observability Access Manager), a single dashboard in the monitoring account can additionally:

- Graph metrics from multiple source accounts and Regions in one widget — even one metric-math expression mixing sources.
- Create alarms that watch metrics living in a source account.
- Run one Logs Insights query across log groups in multiple source accounts.
- Show source-account nodes in an X-Ray trace map, filterable by account.

This is done in JSON by adding `accountId` (and optionally `region`) at either the **widget** level (applies to every metric in it) or the **individual metric** level (mixing accounts/Regions within one graph or metric-math expression):

```json
{
  "metrics": [
    [{ "expression": "SUM(METRICS())", "id": "e1", "label": "Total" }],
    ["AWS/EC2", "CPUUtilization", { "id": "m1", "accountId": "111122223333", "region": "us-east-1" }],
    [".", ".", { "id": "m2", "accountId": "444455556666", "region": "eu-west-1" }]
  ],
  "view": "timeSeries",
  "region": "us-east-1"
}
```

In the console, the fastest path is: from the monitoring account, add metrics from other accounts/Regions to a graph via **Metrics → Classic metrics** (switch account/Region per addition), then **Actions → Add to dashboard**. A signed-in monitoring account shows a blue **Monitoring account** badge on every page where this applies. Note that automatic (home-page) dashboards stay account- **and Region-**scoped no matter what, even for a monitoring account — both cross-Region and cross-account views only exist on custom dashboards you deliberately build.

### Dashboard variables

A dashboard variable lets one dashboard swap its target (a Lambda function name, an instance ID, a Region) via an input field, instead of duplicating the dashboard per target — this both reduces maintenance and, since fewer dashboards means fewer $3/month line items, reduces cost.

- **Property variable** — replaces every instance of a chosen JSON property (any dimension name like `FunctionName`/`InstanceId`, or a structural field like `region`) across all widgets at once. Handles most use cases and is the simpler of the two to set up.
- **Pattern variable** — uses a regular expression to swap only part of a JSON property's value, useful when you need to transform a value rather than substitute it wholesale (e.g. switching Region codes embedded inside an ARN).

Variables can be copied from one dashboard to another once defined. **Caveat:** if you share a dashboard that uses variables, viewers cannot change the variable's value — they see whatever it was set to when shared.

### Sharing dashboards

Three sharing modes exist, none of which require the viewer to have an AWS account:

1. **Named viewers** — up to 5 email addresses per dashboard; each recipient sets their own password to view it.
2. **Public link** — anyone with the URL can view the dashboard, no login at all.
3. **SSO-wide sharing** — a SAML-based third-party SSO provider (integrated via Amazon Cognito) is granted access to every dashboard in the account for all its members.

**Security implications are the important part here.** Sharing (by any of the three modes) provisions an IAM role in the account granting the viewer `cloudwatch:GetMetricData`, `cloudwatch:DescribeAlarms`, `cloudwatch:GetInsightRuleReport`, and `ec2:DescribeTags` — and AWS explicitly documents that `GetMetricData` and `DescribeTags` **cannot be scoped down** to just the dashboard's own metrics/instances. In practice: anyone with a public dashboard link can query *any* CloudWatch metric and see the tags on *any* EC2 instance in the account, not just what's rendered on the dashboard. Treat "share publicly" as "grant read access to this account's telemetry," not "share this one page."

Several widget types are **hidden from shared viewers by default** and require explicitly extending the auto-created sharing IAM role's policy to expose them: composite alarm widgets (need `cloudwatch:DescribeAlarms` with `Resource: "*"`, ideally paired with a `Deny` on sensitive alarm ARNs), Logs Insights table widgets (need `logs:StartQuery`/`GetQueryResults`/etc., scoped to specific log group ARNs), custom (Lambda-backed) widgets (need `lambda:InvokeFunction` scoped to specific function ARNs), and PromQL-query widgets (need `cloudwatch:ListMetrics`). Metric widgets with alarm annotations also render blank for shared viewers regardless of policy.

Sharing itself is free; the widgets inside a shared dashboard still incur normal CloudWatch API/query charges when rendered. Sharing resources (Cognito user/identity pools) are always created in `us-east-1` regardless of the dashboard's own Region, and AWS explicitly warns against manually modifying the auto-generated Cognito/IAM resources.

### Pricing & free tier

- **Free tier:** 3 custom dashboards, each referencing up to 50 metrics, per month.
- **Beyond free tier:** **$3 per custom dashboard per month**, prorated hourly — create/delete mid-month and you're billed only for the hours it existed.
- **Automatic dashboards are always free**, unconditionally.
- Widgets that run Logs Insights queries or invoke a custom (Lambda) source incur their own separate charges on top of the dashboard fee.

## Gotchas worth remembering

- **A public dashboard link is an account-wide read grant, not a scoped view** — `GetMetricData`/`DescribeTags` can't be restricted to the dashboard's own content.
- **Some widget types are silently hidden from shared viewers** (composite alarms, Logs Insights, custom widgets, PromQL, alarm-annotated metric widgets) until you manually extend the sharing role's policy — a common "why can't my viewer see this widget" surprise.
- **Shared dashboards freeze variable values** — don't build a shared "pick your Lambda function" dashboard expecting viewers to switch targets.
- **Automatic (home-page) dashboards never show cross-account data**, even signed into a monitoring account — cross-account viewing only exists on custom dashboards you deliberately build with `accountId`/`region` fields.
- **Fewer, variable-driven dashboards beat many near-duplicate ones** — both operationally and on the $3/dashboard/month line item.
- **"Managed" data connectors (Query Studio's PromQL sources, Multi source query) still run on Lambda under the hood** — AWS templates the deployment via CloudFormation so you don't write the function yourself, but the same "who's invoking this and with what permissions" caution that applies to Custom widgets applies here too, just with the IAM/Secrets Manager wiring done for you.
- **The console's own UI is sometimes ahead of AWS's published docs** — the Text/image widget's image support and dashboard-deep-link syntax aren't in AWS's public Markdown reference page as of this writing, but are confirmed working via the console's own prefilled widget template. Trust what the console shows over a docs page that hasn't caught up yet.

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md)
- [CloudWatch Metrics & Alarms](metrics-and-alarms.md)
- [CloudWatch Application & Infrastructure Observability](application-observability.md)
- [CloudWatch Automation, Governance & Ingestion](automation-and-governance.md)

## References

1. "Using Amazon CloudWatch dashboards" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Dashboards.html>
2. "Getting started with CloudWatch automatic dashboards" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/GettingStarted.html>
3. "Using widgets on CloudWatch dashboards" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-and-work-with-widgets.html>
4. "Sharing CloudWatch dashboards" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-dashboard-sharing.html>
5. "Creating flexible CloudWatch dashboards with dashboard variables" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_dashboard_variables.html>
6. "Dashboard Body Structure and Syntax" — Amazon CloudWatch API Reference. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/CloudWatch-Dashboard-Body-Structure.html>
7. "Amazon CloudWatch Pricing" — AWS product page. <https://aws.amazon.com/cloudwatch/pricing/>
8. "Using Markdown in the Console" — AWS Management Console Getting Started Guide. <https://docs.aws.amazon.com/awsconsolehelpdocs/latest/gsg/aws-markdown.html> (does not yet document image syntax or dashboard deep-links, both confirmed instead via the console's own Text-widget template)
9. "Running PromQL queries in Query Studio" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-PromQL-QueryStudio.html>
10. "Amazon CloudWatch introduces PromQL querying with Query Studio Preview" — AWS What's New, April 2026. <https://aws.amazon.com/about-aws/whats-new/2026/04/amazon-cloudwatch-query-studio-preview>
11. "Connect to a prebuilt data source with a wizard" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_MultiDataSources-Connect.html> (fetched and read in full for the Azure Monitor subsection — no mention of Log Analytics/KQL/Azure AD/Entra ID/sign-in/audit logs anywhere on the page)
12. "Create a Microsoft Entra application and service principal that can access resources" — Microsoft Entra documentation, linked from the AWS page above. <https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal> (AWS's own docs don't state the exact Azure RBAC role required — check this page before granting broader permissions than needed)

---
*Last verified 2026-09-27 against the sources above. AWS documentation changes frequently — re-verify version-specific limits and pricing at docs.aws.amazon.com and aws.amazon.com/cloudwatch/pricing before relying on them.*
