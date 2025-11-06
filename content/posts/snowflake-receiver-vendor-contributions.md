+++
title = 'Why Your Snowflake Monitoring is Two Hours Behind: The Reality of Vendor-Contributed OpenTelemetry Receivers'
date = 2025-11-06T00:00:00-07:00
draft = false
tags = ['opentelemetry', 'snowflake', 'observability', 'otel', 'monitoring', 'data-engineering']
description = 'The OpenTelemetry Collector Contrib Snowflake receiver uses ACCOUNT_USAGE schema with 45-minute to 3-hour latency. Here is why vendor contributions often miss production requirements and how to build receivers that actually work.'
+++

## When "OpenTelemetry Support" Does Not Mean Production-Ready Monitoring

If you have migrated from DataDog or Dynatrace to OpenTelemetry and are using the official Snowflake receiver from the `opentelemetry-collector-contrib` repository, you may have noticed something troubling: your dashboards show you problems that happened hours ago, not minutes ago.

The issue? The official Snowflake receiver queries the `ACCOUNT_USAGE` schema exclusively, which has a documented 45-minute to 3-hour data latency for most views.

For a data warehouse handling millions of dollars in monthly compute costs, that is not monitoring—that is post-incident forensics.

## The ACCOUNT_USAGE Trap

Snowflake provides two primary schemas for system metadata and performance information: `ACCOUNT_USAGE` and `INFORMATION_SCHEMA`. They serve different purposes and have dramatically different latency characteristics.

**ACCOUNT_USAGE Schema:**
- Latency: 45 minutes to 3 hours for most views
- Retention: Up to 365 days of historical data
- Coverage: Comprehensive billing, storage, and long-term usage patterns
- Use case: Historical analysis, cost attribution, compliance reporting

**INFORMATION_SCHEMA Schema:**
- Latency: Near real-time (seconds to minutes)
- Retention: Limited to recent data (hours to days depending on view)
- Coverage: Current operational state and recent query activity
- Use case: Performance monitoring, incident response, operational dashboards

The OpenTelemetry Collector Contrib Snowflake receiver exclusively uses `ACCOUNT_USAGE` for all its queries. This design choice makes it fundamentally unsuitable for operational monitoring, regardless of how you configure collection intervals.

## What This Means in Production

Consider a typical incident scenario at an enterprise running Snowflake:

At 2:00 PM, a poorly optimized query starts consuming excessive warehouse resources, blocking other queries and driving up costs. Your team needs to detect this issue, identify the problematic query, and intervene quickly.

With DataDog or Dynatrace Snowflake integrations, which query both `INFORMATION_SCHEMA` and `ACCOUNT_USAGE` appropriately, you would see the issue within minutes. Alerts would fire, dashboards would show the spike, and you could take corrective action.

With the OpenTelemetry Collector Contrib Snowflake receiver, you would not see the problem in your monitoring system until somewhere between 2:45 PM and 5:00 PM—long after the incident has either resolved itself or caused significant business impact.

This latency gap is not a configuration problem. It is an architectural limitation of the receiver's data source selection.

## How Proprietary Integrations Handle Snowflake

DataDog and Dynatrace understand Snowflake's dual schema architecture and use the appropriate data source for each metric category.

**Real-time operational metrics** (query performance, warehouse load, session activity):
- Source: `INFORMATION_SCHEMA` views
- Example views: `QUERY_HISTORY`, `WAREHOUSE_LOAD_HISTORY`, `SESSIONS`
- Latency: Seconds to minutes
- Collection frequency: Every 1-5 minutes

**Historical and billing metrics** (credit consumption, storage usage, long-term trends):
- Source: `ACCOUNT_USAGE` views
- Example views: `WAREHOUSE_METERING_HISTORY`, `STORAGE_USAGE`, `AUTOMATIC_CLUSTERING_HISTORY`
- Latency: 45 minutes to 3 hours
- Collection frequency: Every 15-30 minutes

This architectural split ensures that operational dashboards show current reality while historical analysis and cost attribution leverage the comprehensive retention of `ACCOUNT_USAGE`.

The OpenTelemetry Collector Contrib Snowflake receiver does not make this distinction. Everything goes through `ACCOUNT_USAGE`, producing a monitoring system that fails its primary purpose: telling you what is happening right now.

## The Vendor Contribution Problem

When observability vendors tout their status as "lead contributors" to the OpenTelemetry project, they highlight their commitment to open standards and vendor-neutral telemetry. This is genuinely valuable for the ecosystem.

However, the quality of these contributions varies dramatically. Some receivers in the `opentelemetry-collector-contrib` repository represent production-grade engineering. Others, like the Snowflake receiver, appear to be minimum viable implementations designed to check a marketing box rather than solve actual operational problems.

The Snowflake receiver's exclusive reliance on `ACCOUNT_USAGE` suggests it was built by engineers who understood Snowflake's API surface but did not understand real-world monitoring requirements. No production operations team would design a monitoring system with 2-hour latency if alternatives existed—yet that is exactly what this receiver delivers.

This pattern repeats across multiple contributed receivers. Vendors contribute code that demonstrates technical capability but falls short of operational excellence. The OpenTelemetry project accepts these contributions because they expand ecosystem coverage, even when the implementations have fundamental limitations.

The result? Enterprises migrating from commercial APM solutions discover that "OpenTelemetry support" does not guarantee parity with proprietary integrations. Some receivers work brilliantly. Others, like Snowflake, require complete reimplementation to meet production standards.

## Building a Production-Grade Snowflake Receiver

I built a custom Snowflake receiver that addresses these limitations by querying the appropriate schema for each metric category. Here is the architectural approach:

### Schema-Based Metric Categorization

The receiver classifies metrics into three categories based on operational requirements:

**Category 1: Real-time operational metrics** (1-minute collection interval)
- Query performance and execution statistics
- Warehouse load and queue depth
- Active session counts and blocking relationships
- Source: `INFORMATION_SCHEMA` with 1-hour time windows

**Category 2: Near-time usage metrics** (5-minute collection interval)
- Recent credit consumption by warehouse
- Query history with execution plans
- Task execution status and failures
- Source: Mix of `INFORMATION_SCHEMA` and recent `ACCOUNT_USAGE` data

**Category 3: Historical and billing metrics** (30-minute collection interval)
- Long-term storage trends
- Monthly credit consumption patterns
- Data transfer and replication costs
- Source: `ACCOUNT_USAGE` views with 24-hour windows

This categorization ensures that operational dashboards show current reality while preserving access to historical data for capacity planning and cost attribution.

### Parallel Query Execution

The receiver queries multiple schemas and metric categories in parallel with independent timeout controls. This design prevents slow historical queries from blocking real-time metric collection.

```go
// Simplified example of parallel metric collection
func (s *snowflakeScraper) collectMetrics(ctx context.Context) {
    var wg sync.WaitGroup
    
    // Real-time metrics with tight timeout
    wg.Add(1)
    go func() {
        defer wg.Done()
        ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
        defer cancel()
        s.collectRealtimeMetrics(ctx)
    }()
    
    // Historical metrics with longer timeout
    wg.Add(1)
    go func() {
        defer wg.Done()
        ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
        defer cancel()
        s.collectHistoricalMetrics(ctx)
    }()
    
    wg.Wait()
}
```

Each metric category operates independently. If `ACCOUNT_USAGE` queries run slowly or time out, operational metrics continue flowing without interruption.

### Selective Metric Enablement

Not every deployment needs every metric. The receiver supports granular configuration:

```yaml
receivers:
  snowflake:
    # Real-time operational metrics
    current_queries:
      enabled: true
      interval: "1m"
    
    warehouse_load:
      enabled: true
      interval: "1m"
    
    # Historical analysis
    credit_usage:
      enabled: true
      interval: "5m"
    
    storage_metrics:
      enabled: true
      interval: "30m"
    
    # Optional advanced features
    event_tables:
      enabled: false
    
    replication_usage:
      enabled: false
```

This configuration flexibility reduces cardinality and collection overhead for environments that only need specific metric categories.

### Built with OpenTelemetry Collector Builder

The receiver compiles into a custom collector binary using OCB (OpenTelemetry Collector Builder):

```yaml
dist:
  name: otelcol-snowflake
  description: OpenTelemetry Collector with production-grade Snowflake receiver
  output_path: ./otelcol-snowflake
  otelcol_version: 0.135.0

receivers:
  - gomod: github.com/npcomplete777/snowflakereceiver v0.0.1
    path: ./snowflakereceiver

exporters:
  - gomod: go.opentelemetry.io/collector/exporter/otlphttpexporter v0.135.0

processors:
  - gomod: go.opentelemetry.io/collector/processor/batchprocessor v0.135.0
```

This approach creates a self-contained collector binary that includes exactly the components needed for Snowflake monitoring, with no unnecessary dependencies.

## Why This Matters for Enterprise Migrations

Many Fortune 500 enterprises are evaluating moves away from expensive DataDog or Dynatrace contracts. These migrations hinge on achieving observability parity with existing tools.

When you are monitoring a Snowflake environment handling petabytes of data and spending millions annually on compute, "average query latency is up" alerts that fire two hours after the incident are useless. You need dimensional context in real-time:

- Which specific query pattern is slow
- In which warehouse and database
- What user or service account initiated it
- How it correlates with recent schema changes or deployments

The OpenTelemetry Collector Contrib Snowflake receiver cannot provide this context because it queries historical data sources. A production-grade receiver querying `INFORMATION_SCHEMA` can.

This is why DBMarlin exists as a specialized Snowflake monitoring product despite Dynatrace and DataDog dominating the broader APM market. They saw the gap between what standard monitoring provides and what Snowflake operations actually require.

This custom receiver takes the same approach but outputs to open standards instead of another proprietary platform. Your Snowflake operational metrics can flow to ClickHouse, Honeycomb, Dash0, or any other backend that speaks OTLP—your choice, not your vendor's.

## The Broader Implication for OpenTelemetry Contributions

The Snowflake receiver's limitations raise an uncomfortable question about vendor contributions to OpenTelemetry: what quality standards should the project enforce?

The current approach prioritizes ecosystem coverage over implementation excellence. Vendors can contribute receivers with fundamental architectural flaws, and the project accepts them because having some Snowflake support is better than having none.

This creates a hidden cost for enterprises adopting OpenTelemetry. They discover that "supported" does not mean "production-ready," and they must evaluate each receiver individually to determine if it actually solves their operational problems.

For receivers like Snowflake that interact with complex systems offering multiple data sources with different latency characteristics, this evaluation requires deep technical expertise. Most organizations attempting OpenTelemetry migrations do not have this expertise in-house and end up discovering limitations only after deployment.

The OpenTelemetry project would benefit from more rigorous contribution standards that distinguish between "proof of concept" receivers and "production-grade" implementations. Until then, enterprises should approach vendor-contributed receivers with healthy skepticism and invest in thorough testing before relying on them for critical operations.

## Where to Go From Here

The observability ecosystem is maturing, but significant gaps remain between marketing claims and operational reality. Building custom receivers that address these gaps is how enterprises achieve both cost savings and operational excellence.

For Snowflake specifically, the path forward is clear:

**If you only need historical analysis and cost attribution:** The OpenTelemetry Collector Contrib Snowflake receiver works fine. Accept the 2-hour latency as a reasonable trade-off for comprehensive historical data access.

**If you need operational monitoring:** Query `INFORMATION_SCHEMA` directly for real-time metrics and use `ACCOUNT_USAGE` only for historical and billing data. This requires building a custom receiver or modifying the contrib version.

**If you need operational monitoring without vendor lock-in:** Build a custom OpenTelemetry receiver that implements proper schema selection and exports to OTLP. Your dimensional data can then flow to any backend that handles high-cardinality metrics well.

The real value is not just using OpenTelemetry—it is collecting the right data from the right sources at the right time, then having the freedom to route it wherever your architecture demands.

When vendors claim "OpenTelemetry support," ask what that actually means. Support for what? With what latency characteristics? Preserving which dimensions? The answers reveal whether their contribution solves real problems or just checks marketing boxes.

## References

- [Snowflake ACCOUNT_USAGE vs INFORMATION_SCHEMA Documentation](https://docs.snowflake.com/en/sql-reference/account-usage)
- [OpenTelemetry Collector Contrib Snowflake Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/snowflakereceiver)
- [Custom Snowflake Receiver GitHub Repository](https://github.com/npcomplete777/snowflakereceiver)
- [OpenTelemetry Collector Builder Documentation](https://opentelemetry.io/docs/collector/custom-collector/)
- [Snowflake Query History Latency](https://docs.snowflake.com/en/sql-reference/latency)
