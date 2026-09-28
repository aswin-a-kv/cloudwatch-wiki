---
title: CloudWatch Settings & S3 Tables Integration
summary: Account-level CloudWatch console settings (resource tags for telemetry, alarm mute rules) and the S3 Tables integration for querying CloudWatch Logs data with Athena/Redshift/EMR via Apache Iceberg
last_verified: 2026-09-21
categories: [Amazon Web Services, Cloud monitoring, Observability, Data lakes]
---

# CloudWatch Settings & S3 Tables Integration

The CloudWatch console has a dedicated **Settings** page for account-wide configuration that doesn't belong to any single sub-service — telemetry enrichment, alarm muting schedules, and (newest) a managed export of CloudWatch Logs data into **Amazon S3 Tables**, AWS's managed Apache Iceberg table store, for querying with external analytics engines. This article is the practical companion to [Amazon CloudWatch](amazon-cloudwatch.md), covering what lives on that Settings page and how the S3 Tables integration works end to end.

## Resource tags for telemetry

By default, CloudWatch metrics and log events carry AWS resource identifiers (instance IDs, ARNs) but not the resource's **tags** — so a query like "show me all Lambda duration metrics for `team:payments`" isn't directly possible without joining against tag data elsewhere. **Resource tags for telemetry** closes that gap: once enabled (CloudWatch console → **Settings** → toggle **Enable resource tags for telemetry**, or `aws observabilityadmin start-telemetry-enrichment`), CloudWatch uses **AWS Resource Explorer** to build an account-wide index of resources and their tags, then enriches infrastructure metrics and log events with the matching tags as they arrive. Discovery of existing tags can take up to 3 hours after enabling.

This unlocks tag-based grouping/filtering in metric queries and dashboards, and — the more interesting use case — **dynamic alarms**: an alarm scoped by tag (e.g. `team:payments`) automatically picks up new resources carrying that tag without anyone editing the alarm. The feature is free; the underlying Resource Explorer index and managed view it creates are billed under their own (also currently free-tier) pricing.

Required permissions to enable it: `observabilityadmin:StartTelemetryEnrichment`, `iam:CreateServiceLinkedRole`, `resource-explorer-2:CreateIndex`, `resource-explorer-2:CreateManagedView`, `resource-explorer-2:CreateStreamingAccessForService`.

## Alarm mute rules

Also configured from **Alarms → Mute Rules** in the console (a sibling to the Settings page, functionally part of the same "account-level alarm behavior" surface): a mute rule silences a set of alarms — no state-change notifications fire — for a scheduled window, without disabling the alarm or losing its evaluation history. Two shapes:

- **Scheduled mute rule** — one-time (start/end date+time in a chosen timezone) or recurring, either built via the console's schedule picker (repeat daily/weekly/on specific days, optional expiry) or a raw cron expression plus a duration. Targets a chosen set of alarms and can carry tags.
- **Quick mute** — select alarms in the Alarms list → **Actions → Mute** → pick a preset window (15 min / 1 h / 3 h) or a custom "mute until" time. Meant for "I know this alarm will fire during my maintenance window, silence it now" rather than a standing schedule.

Existing mute rules can also be applied retroactively to newly created alarms (**Actions → Mute → Apply existing mute rules**). Under the hood this is the `PutAlarmMuteRule` API.

## S3 Tables Integration (CloudWatch Logs)

The **S3 Tables Integration** makes CloudWatch Logs data queryable from external analytics engines — Amazon Athena, Amazon Redshift, Amazon EMR, and any third-party tool that speaks Apache Iceberg — without CloudWatch Logs Insights being the only way in. It complements, rather than replaces, [Metric Streams](automation-and-governance.md#metric-streams): Metric Streams pushes **metrics** continuously to an external destination (typically Firehose) for real-time forwarding to third-party observability vendors; S3 Tables Integration exposes **logs** at rest as a managed, query-in-place Iceberg table for ad hoc SQL analytics and correlation with non-CloudWatch data. Neither one is a general export mechanism for the other's data type.

### How it works

Enabling the integration creates a CloudWatch-managed **S3 table bucket** named `aws-cloudwatch`. You then **associate** specific log data sources (by source name and type) with the integration — either accept every current and future data source in the account (the default), or opt in selectively. Once associated, new log events matching that source are continuously delivered into the corresponding Iceberg table under the `logs` namespace in that table bucket.

Key mechanics:

- **No backfill.** Only log events ingested *after* the association is created flow to S3 Tables — existing log history is not retroactively copied.
- **Retention mirrors the log group.** Data in the S3 table follows the same retention policy as the source log group; when the log group's retention expires (or the group/stream is deleted), CloudWatch removes the corresponding data from the S3 table too. This makes it unsuitable as a long-term archive independent of the log group's own retention setting.
- **No extra storage/maintenance cost.** AWS states there's no additional storage or Iceberg table-maintenance (compaction, etc.) charge beyond existing CloudWatch Logs ingestion/storage pricing — you pay only for the queries run against the data (e.g. Athena's per-TB-scanned pricing).

### Setup steps

1. **Create the integration** — CloudWatch console → **Settings** → **Global** → **Create S3 Table Integration**. Choose the encryption settings for the S3 table data and the IAM role CloudWatch Logs will assume to write into it. Leave **Enable all log sources and types to be available in the S3 table** checked to auto-associate everything (including future sources), or uncheck it to associate sources individually afterward via **Settings → Global → Manage S3 Table Integration**, or per-source from **Log Management → Data Sources → Data source actions → Associate with S3 Tables Integration**.
2. **IAM for the setup principal** — the user/role creating the integration needs `observabilityadmin:CreateS3TableIntegration`, `logs:AssociateSourceToS3TableIntegration`, and `s3tables:CreateTableBucket` / `s3tables:PutTableBucketEncryption` / `s3tables:PutTableBucketPolicy`.
3. **IAM for the service role** — the role CloudWatch Logs assumes to write data needs a policy granting `logs:integrateWithS3Table` scoped to the source log group's ARN, plus a trust policy allowing `logs.amazonaws.com` to assume it, scoped by `aws:SourceAccount`/`aws:SourceArn` conditions.
4. **KMS (if the log group uses a customer-managed key)** — grant `kms:DescribeKey`/`kms:GenerateDataKey`/`kms:Decrypt` to the `systemtables.cloudwatch.amazonaws.com` principal, and `kms:GenerateDataKey`/`kms:Decrypt` to `maintenance.s3tables.amazonaws.com`, scoped by the table/table-bucket ARN.
5. **Enable the S3 Tables ↔ analytics integration** — in the S3 console, **Table buckets → Enable integration**, which (on first use per Region) creates a service-linked role letting AWS Lake Formation federate access to the table bucket's tables in the Glue Data Catalog.
6. **Grant query access via Lake Formation** — CloudWatch Logs writing to the table does not by itself grant anyone permission to *read* it. In the Lake Formation console, **Data lake permissions → Grant**, select the IAM principals that should query the data, target the `aws-cloudwatch` table bucket (optionally a specific table), and grant **Select** + **Describe**. This must be repeated per principal.
7. **Query it** — in Athena, pick the **Amazon S3 Tables** catalog from the data source dropdown (only visible once Lake Formation permissions are granted); the associated log tables appear as regular databases/tables, queryable with standard SQL, e.g. `SELECT * FROM "amazon_vpc__flow" LIMIT 100;`.

### When to reach for it

Use S3 Tables Integration when you need SQL-style analysis beyond what Logs Insights' query language offers, want to join log data against non-CloudWatch datasets in Athena/Redshift/EMR, or want to hand log data to a data/analytics team that already works in those tools rather than the CloudWatch console. It is not a substitute for Logs Insights' native filter/`stats` querying for day-to-day operational use, and it is not a durable long-term log archive on its own — retention still tracks the source log group.

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md) — the overview article this one extends
- [CloudWatch Logs](logs.md) — log groups, retention classes, and Logs Insights, which S3 Tables Integration complements
- [CloudWatch Automation, Governance & Ingestion](automation-and-governance.md) — Metric Streams (the metrics-side counterpart to S3 Tables Integration's logs-side export) and Telemetry Configuration (Ingestion), the sibling `observabilityadmin` feature to resource tags for telemetry

## References

1. "Access logs with S3 Tables Integration" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/s3-tables-integration.html>
2. "Integrating Amazon S3 Tables with AWS analytics services" — Amazon S3 User Guide. <https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-aws.html>
3. "Resource tags for telemetry" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/resource-tags-for-telemetry.html>
4. "Enable resource tags on telemetry" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/EnableResourceTagsOnTelemetry.html>
5. "Configure alarm mute rules" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-mute-rules-configure.html>
6. "PutAlarmMuteRule" — Amazon CloudWatch API Reference. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutAlarmMuteRule.html>

---
*Last verified 2026-09-21 against the sources above. This is a newer/actively evolving area of CloudWatch (S3 Tables Integration, resource tags for telemetry) — re-verify console navigation and IAM action names before relying on them, as they are more likely to shift than long-standing CloudWatch features.*
