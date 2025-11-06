+++
title = 'Beyond Prometheus: Building a High-Cardinality CockroachDB Receiver for OpenTelemetry'
date = 2024-10-31T00:00:00-07:00
draft = false
tags = ['opentelemetry', 'cockroachdb', 'observability', 'otel', 'monitoring', 'high-cardinality']
description = 'How to build a custom OpenTelemetry receiver that queries CockroachDB internals directly, preserving high-cardinality dimensional data that standard Prometheus scraping loses'
+++

## Why Standard Monitoring Falls Short for CockroachDB

If you are familiar with CockroachDB performance, you have probably noticed that monitoring solutions take the easy path: they scrape the `/_status/vars` endpoint and call it a day.

DataDog uses an OpenMetrics-based integration (a modern parsing framework for Prometheus-format metrics) that still collects from `/_status/vars`. Grafana uses Prometheus or Grafana Agent to scrape the same endpoint. Both get the same pre-aggregated, infrastructure-level data.

Here is the problem: The `/_status/vars` endpoint only exposes metrics like CPU usage, memory consumption, and query throughput—cluster-wide averages that are great for keeping the lights on, but terrible for actually understanding what is happening inside your database at the query, session, or transaction level.

When a production incident hits and you need to know which specific queries are causing contention, what sessions are blocking others, or why a particular table's performance degraded, those aggregated metrics leave you flying blind. You have lost all the dimensional context that operations teams need to troubleshoot real issues.

DataDog's self-hosted integration explicitly states it "only supports displaying cluster-wide averages of reported metrics" with no per-node filtering. That is a fundamental limitation of the data source, not the collection mechanism.

## The DBMarlin Approach: Going Straight to the Source

Only one vendor in the observability space has figured this out: DBMarlin. Instead of scraping the `/_status/vars` endpoint, DBMarlin queries CockroachDB's `crdb_internal` system catalog directly via SQL. This system catalog contains the real operational treasure trove:

- `statement_statistics` - SQL fingerprints with execution plans and latency breakdowns
- `cluster_locks` - Real-time lock contention data by transaction
- `cluster_sessions` - Active session states with query details
- `table_spans` - Actual table and index usage statistics
- `ranges` - Replica health and distribution information

This approach preserves all the high-cardinality dimensional data that gets lost in pre-aggregated metrics.

## Understanding Collection Methods: OpenMetrics vs. SQL Queries

When DataDog advertises their "OpenMetrics-based integration," they are describing how their agent parses the Prometheus format metrics—not what data they collect. Think of it this way:

**OpenMetrics-Based Collection (DataDog, Grafana):**

- Collection Framework: Modern OpenMetrics parser or Prometheus scraper
- Data Source: HTTP endpoint `/_status/vars`
- Data Format: Pre-aggregated time-series in Prometheus/OpenMetrics text format
- Content: Infrastructure metrics (CPU, memory, query counts)
- Result: Cluster-wide averages, no dimensional context

**SQL-Based Collection (DBMarlin, Custom OTel Receivers):**

- Collection Framework: Database driver executing SQL queries
- Data Source: `crdb_internal` system catalog tables via PostgreSQL protocol
- Data Format: Relational result sets from internal tables
- Content: Operational metrics (per-query stats, session details, lock contention)
- Result: High-cardinality dimensional data with full business context

DataDog's "latest mode" (`openmetrics_endpoint`) and "legacy mode" (`prometheus_url`) both hit the same `/_status/vars` endpoint—they just use different parsing engines. You are still getting the same limited data either way.

The distinction matters: if you want to see which specific query is slow, in which database, with what lock contention, you need to query `crdb_internal` directly. No amount of sophisticated metric parsing will give you data that is not there.

## Why OpenTelemetry Changes Everything

For enterprises migrating from commercial APM solutions like DataDog to open-source observability stacks, this dimensional data problem is make-or-break. Teams are used to deep, contextual visibility into their databases.

This is where OpenTelemetry receivers come in. Think of receivers as similar to the proprietary "integrations" or "extensions" you would use in DataDog—modular components that know how to collect data from specific sources. The difference? OpenTelemetry receivers are vendor-neutral, extensible, and do not lock you into a specific backend.

Just like a DataDog integration might scrape an API or execute SQL queries to collect metrics, an OpenTelemetry receiver does the same thing, but outputs data in a standardized format that can be sent to any backend: Prometheus, ClickHouse, Chronosphere, Honeycomb, and others.

## Building a Custom CockroachDB Receiver

I built a custom OpenTelemetry receiver that takes the DBMarlin approach—querying `crdb_internal` directly to preserve all dimensional context. Here is what makes it different:

### High-Cardinality Dimensional Metrics

Instead of aggregated counters, every metric includes rich dimensions:

- Query fingerprints with database, schema, and user context
- Session metrics tagged with application name and node location
- Transaction contention data with blocking/blocked relationships
- Index usage per table with read/write patterns

### Built with OpenTelemetry Collector Builder (OCB)

The receiver is compiled into a custom collector binary using OCB, OpenTelemetry's official tool for building modular collectors. Here is the process:
```yaml
# Define your collector components in builder-config.yaml
dist:
  name: otelcol-cockroachdb
  description: OpenTelemetry Collector with CockroachDB receiver
  
receivers:
  - gomod: github.com/npcomplete777/cockroachdbreceiver/cockroachreceiver v0.0.1
    path: ./cockroachreceiver
    
exporters:
  - gomod: go.opentelemetry.io/collector/exporter/otlphttpexporter v0.136.0
```
```bash
# Build the custom collector
~/ocb --config builder-config.yaml

# Run with your configuration  
./otelcol-cockroachdb/otelcol-cockroachdb --config config.yaml
```

OCB handles all the complexity of dependency resolution, component registration, and binary compilation. You define what components you need; OCB builds a collector that includes exactly those components—nothing more, nothing less.

### Parallel Query Execution

The receiver queries multiple `crdb_internal` tables in parallel:

- Query performance metrics
- Active session statistics
- Lock contention analysis
- Range distribution health
- Table and index usage patterns

Each metric category runs independently with configurable timeouts, ensuring that one slow query does not block the entire collection cycle.

### Selective Metrics Collection

Not every environment needs every metric. The receiver supports selective enabling:
```yaml
receivers:
  cockroachdb:
    enabled_metrics:
      - query       # SQL statement statistics
      - session     # Active connections
      - contention  # Lock conflicts
```

This reduces cardinality when needed while preserving the option for deep visibility when troubleshooting.

## Why This Matters for Enterprise Migrations

Many Fortune 500 enterprises are evaluating moves away from expensive annual DataDog contracts. But they cannot sacrifice visibility. The observability data needs to be as good or better than what they had.

When you are monitoring a distributed SQL database that is handling thousands of transactions per second across multiple regions, you need dimensional context. "Average query latency is up" does not cut it. You need to know:

- Which query fingerprint is slow
- In which database and region
- What is blocking it
- How it correlates with recent deployments

An OpenMetrics integration scraping `/_status/vars` cannot answer those questions—whether it is running in DataDog, Grafana, or a custom Prometheus setup. The data simply is not there. A receiver querying `crdb_internal` can.

This is why DBMarlin exists as a separate product despite DataDog dominating the market: they saw the gap and filled it. This custom receiver does the same thing, but outputs to open standards instead of another proprietary platform.

## The Receiver vs. Integration Mental Model

If you are coming from the proprietary APM world, here is how to think about it:

**DataDog Integration:**

- Vendor-specific code that collects from a data source
- Can scrape HTTP endpoints (like `/_status/vars`) OR execute SQL queries (depending on integration)
- Outputs to vendor's proprietary format
- Locked into that vendor's backend
- Configuration managed in vendor's UI

**OpenTelemetry Receiver:**

- Vendor-neutral code that collects from a data source
- Can scrape HTTP endpoints OR execute SQL queries OR call APIs (depending on receiver implementation)
- Outputs to OTLP (OpenTelemetry Protocol) standard
- Send to any backend that speaks OTLP
- Configuration as code, version-controlled
- Build custom collectors with OCB for your exact needs

The key insight: Proprietary integrations and OTel receivers can collect data in similar ways. DataDog has integrations that execute SQL queries (for Oracle, SQL Server). This custom CockroachDB receiver also executes SQL queries. The difference is not how data is collected—it is:

- **Vendor lock-in** - DataDog integration outputs only work with DataDog; OTel receivers work with any OTLP backend
- **Extensibility** - You can build custom OTel receivers for your specific needs; proprietary integrations require vendor support
- **Data ownership** - OTel data flows where you decide; proprietary integrations force your data through vendor infrastructure

The receiver handles the "how do I get data from X source" problem. Your choice of exporter handles the "where does this data go" problem. They are decoupled. That is the power of OpenTelemetry.

## Production Deployment: Kubernetes-Native High Availability

Custom OpenTelemetry collectors deploy as standard Kubernetes Deployments, giving you enterprise-grade high availability out of the box:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otelcol-cockroachdb
spec:
  replicas: 3  # HA across availability zones
  selector:
    matchLabels:
      app: otelcol-cockroachdb
  template:
    spec:
      containers:
      - name: otelcol
        image: your-registry/otelcol-cockroachdb:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
```

Kubernetes handles the operational complexity:

- Replica sets ensure your collectors stay running across node failures
- Horizontal Pod Autoscaling scales collection capacity with query volume
- Rolling updates deploy new receiver versions without collection gaps
- Health checks automatically restart failed collectors

Each collector instance queries CockroachDB independently and exports to your backend—no coordination required. Your observability pipeline scales horizontally just like your database does.

## Where to Go From Here

The open-source observability ecosystem is maturing rapidly. ClickHouse-based backends like Uptrace and SigNoz can handle petabyte-scale, high-cardinality data that would bankrupt you on traditional SaaS platforms. But they need dimensional data to be useful.

Building custom receivers that preserve this context is how enterprises achieve both cost savings and operational excellence.

For CockroachDB specifically, the path is clear:

**If you only need infrastructure metrics:** Use DataDog's OpenMetrics integration or Grafana's Prometheus scraper pointing to `/_status/vars`. You will get cluster health, resource utilization, and basic throughput metrics. This is sufficient for capacity planning and keeping the lights on.

**If you need operational visibility:** Query `crdb_internal` directly (like DBMarlin does) to preserve dimensional context. This is essential for troubleshooting production incidents, understanding query performance, and identifying lock contention.

**If you need operational visibility without vendor lock-in:** Build a custom OpenTelemetry receiver that queries `crdb_internal` and exports to OTLP. Now your dimensional data can flow to ClickHouse, Dash0, or any other backend—your choice, not your vendor's.

The real value is not just using OpenTelemetry—it is collecting the right data in the first place, then having the freedom to route it wherever your architecture demands.

## References

- [CockroachDB Receiver GitHub Repository](https://github.com/npcomplete777/cockroachdbreceiver)
- [CockroachDB Monitoring and Alerting Documentation](https://www.cockroachlabs.com/docs/stable/monitoring-and-alerting)
- [CockroachDB crdb_internal System Catalog](https://www.cockroachlabs.com/docs/stable/crdb-internal)
- [DBMarlin for CockroachDB](https://www.dbmarlin.com/dbmarlin-for-cockroachdb)
- [OpenTelemetry Custom Collector Documentation](https://opentelemetry.io/docs/collector/custom-collector/)
- [DataDog CockroachDB Integration](https://docs.datadoghq.com/integrations/cockroachdb)
