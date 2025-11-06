+++
title = 'Databricks Observability Beyond the API: Building a Two-Tier OpenTelemetry Solution for Workspace and Cluster Monitoring'
date = 2025-11-06T00:00:00-07:00
draft = false
tags = ['opentelemetry', 'databricks', 'apache-spark', 'observability', 'otel', 'monitoring']
description = 'How to achieve comprehensive Databricks monitoring through a custom OpenTelemetry receiver for workspace metrics combined with init script deployment for internal Apache Spark telemetry'
+++

## The Databricks Monitoring Gap

If you are running Databricks workloads at enterprise scale, you have probably discovered that comprehensive observability requires piecing together data from multiple sources that were never designed to work together.

Databricks provides REST APIs for workspace-level metrics: job runs, warehouse states, user activity, and costs. These APIs give you a management console perspective on what happened, but they reveal almost nothing about what is actually happening inside your Spark clusters during execution.

When a job runs slowly or fails, the workspace APIs tell you it failed. They do not tell you which executor ran out of memory, which stage caused the bottleneck, or why shuffle operations took three times longer than usual. For that operational context, you need access to internal Apache Spark metrics—telemetry that exists inside running clusters but is inaccessible to external monitoring systems.

DataDog and Dynatrace solve this problem by deploying agents inside Databricks clusters via init scripts. Their agents scrape local Spark metrics and correlate them with workspace-level data. This approach works, but it locks you into their proprietary platforms and pricing models.

For enterprises migrating to OpenTelemetry, the path forward requires building a two-tier architecture: a custom receiver for workspace APIs combined with init script deployment for cluster-internal metrics. Neither approach alone provides complete visibility.

## Understanding Databricks Observability Architecture

Databricks exposes telemetry through fundamentally different mechanisms depending on what you want to monitor.

### Workspace-Level Metrics (External Access)

The Databricks REST API provides management and operational data accessible from outside your clusters:

**Jobs API:**
- Job run states and durations
- Task execution times
- Queue and setup overhead
- Success rates and failure modes

**Clusters API:**
- Cluster lifecycle events
- Autoscaling activity
- Configuration details

**SQL Warehouses API:**
- Warehouse states and sizes
- Query execution patterns
- Serverless compute metrics

**Workspace API:**
- User and group management
- Token lifecycle
- DBFS storage utilization
- Cluster policies

These APIs operate at the orchestration layer. They tell you what Databricks scheduled, when it ran, and whether it succeeded. They do not reveal how Spark executed the work or where resource contention occurred.

### Cluster-Level Metrics (Internal Access Only)

Apache Spark exposes detailed runtime metrics through its built-in web UI on port 4040. This endpoint provides operational telemetry that external APIs cannot access:

**Executor Metrics:**
- Memory usage per executor (heap, off-heap, storage)
- Active and completed tasks
- Shuffle read and write volumes
- GC time and frequency

**Stage Metrics:**
- Stage duration and task distribution
- Input and output record counts
- Spill statistics

**Job Metrics:**
- DAG scheduler performance
- Job submission and completion times
- Application-level resource consumption

**Streaming Metrics:**
- Processing rates
- Batch durations
- Input rates and backpressure

**Driver Metrics:**
- JVM heap utilization
- BlockManager operations
- LiveListenerBus queue depth

These metrics exist at `http://localhost:4040/metrics/prometheus` inside each cluster, accessible only to processes running on the driver node. External monitoring systems cannot reach this endpoint without deploying agents inside the cluster infrastructure.

This architectural separation creates the fundamental challenge: workspace APIs provide high-level orchestration visibility while cluster-internal metrics provide low-level execution visibility. Complete observability requires accessing both.

## How Proprietary Solutions Handle This

DataDog and Dynatrace understand the workspace versus cluster distinction and implement two-tier monitoring architectures.

### Dynatrace Databricks Extensions

Dynatrace offers two separate extensions that work in harmony:

**Workspace Extension (External):**
- Remotely queries Databricks REST APIs
- Collects job run metrics, cost data, warehouse states
- Works for serverless compute where agents cannot deploy
- Transforms API data into OpenTelemetry traces with jobs as parent spans and tasks as children

**Cluster Extension (Agent-Based):**
- OneAgent deployment via init scripts on cluster nodes
- Scrapes Spark REST API locally from port 4040
- Accesses Ganglia metrics (deprecated in Runtime 13+)
- Captures executor-level resource utilization

The workspace extension provides the "what happened" narrative while the cluster extension provides the "how it happened" operational context. Together, they deliver the comprehensive visibility enterprises expect from a \$30,000+ annual APM contract.

### DataDog Databricks Integration

DataDog follows a similar pattern with their Databricks integration:

**External API Collection:**
- Polls Databricks Jobs API and Clusters API
- Aggregates job run data and cluster lifecycle events
- Correlates with DBU consumption for cost attribution

**In-Cluster Agent:**
- Datadog Agent deployed via init script
- Scrapes Spark UI metrics from localhost:4040
- Collects system-level metrics via Node Exporter integration
- Exports everything to DataDog backend

Both vendors recognize that API-only monitoring leaves massive visibility gaps for production Spark workloads. Their dual-mode approach is fundamentally correct—the implementation just happens to be proprietary and expensive.

## Building a Two-Tier OpenTelemetry Solution

I built a Databricks monitoring system that achieves parity with proprietary integrations using vendor-neutral OpenTelemetry components. The architecture consists of two independent collectors with distinct deployment patterns and data sources.

### Tier 1: Custom Workspace Receiver (External)

The workspace receiver is a custom OpenTelemetry receiver that queries Databricks REST APIs from outside the cluster infrastructure. This component collects 21 workspace-level metrics across four categories.

**Job and Task Metrics:**
- `databricks.jobs.total` - Total configured jobs in workspace
- `databricks.job_runs.count` - Job runs by result state
- `databricks.job.duration.run` - Complete run duration including all phases
- `databricks.job.duration.queue` - Time waiting for cluster availability
- `databricks.job.duration.setup` - Cluster startup and library installation time
- `databricks.job.duration.execution` - Actual Spark job execution time
- `databricks.job.duration.cleanup` - Teardown and result persistence time
- `databricks.job.success_rate` - Success percentage over collection window
- `databricks.job.cost` - Estimated job cost based on DBU consumption
- `databricks.tasks.duration` - Individual task execution time
- `databricks.tasks.setup_duration` - Task initialization overhead

**Infrastructure Metrics:**
- `databricks.warehouses.count` - SQL warehouses by operational state
- `databricks.workspace.objects` - Notebooks, libraries, and artifacts by type
- `databricks.dbfs.storage_bytes` - DBFS utilization for workspace storage
- `databricks.dbfs.file_count` - Total files in DBFS

**Management Metrics:**
- `databricks.tokens.count` - Active personal access tokens
- `databricks.tokens.days_until_expiry` - Token lifecycle tracking
- `databricks.users.count` - Users by active/inactive status
- `databricks.groups.count` - Total workspace groups
- `databricks.policies.count` - Cluster policy count

All metrics include dimensional attributes like `job_name`, `result_state`, `warehouse_state`, and `databricks.host` for filtering and aggregation.

The receiver is built using OpenTelemetry Collector Builder (OCB) and compiles into a standalone binary:

```yaml
dist:
  name: otelcol-databricks
  description: OpenTelemetry Collector with Databricks workspace receiver
  output_path: ./otelcol-databricks
  otelcol_version: 0.135.0

receivers:
  - gomod: github.com/npcomplete777/databricksreceiver v0.0.1
    path: ./receiver

exporters:
  - gomod: go.opentelemetry.io/collector/exporter/otlphttpexporter v0.135.0

processors:
  - gomod: go.opentelemetry.io/collector/processor/batchprocessor v0.135.0
```

This receiver runs continuously outside Databricks infrastructure with a configurable collection interval (typically 60 seconds for API rate limiting considerations). It provides management-level visibility into workspace operations, job scheduling patterns, and cost attribution.

### Tier 2: Cluster-Internal Collection (Init Script)

The cluster-level collector deploys inside Databricks clusters via init scripts—shell scripts that execute during cluster startup before the Apache Spark driver or executor JVMs initialize. This deployment pattern gives the collector localhost access to Spark metrics that external systems cannot reach.

**Init Script Architecture:**

Init scripts run on every cluster node during startup with two critical characteristics that determine the monitoring design:

1. **Lifecycle binding:** Init scripts execute once during cluster creation and remain active for the cluster's lifetime. This means collectors deployed via init script automatically start with clusters and terminate when clusters shut down—perfect for ephemeral Databricks compute patterns.

2. **Node targeting:** Init scripts can target specific node types (driver-only, executor-only, or all nodes), allowing selective deployment of monitoring components based on metric access requirements.

The recommended architecture deploys the OpenTelemetry Collector only on the driver node. Apache Spark aggregates executor metrics at the driver's REST API endpoint, eliminating the need to scrape individual executors directly. This design minimizes network overhead and collector resource consumption while capturing comprehensive telemetry.

**Implementation Details:**

The init script performs these operations during cluster startup:

```bash
#!/bin/bash
# databricks-otel-init.sh

# Only deploy on driver node
if [[ $DB_IS_DRIVER = "TRUE" ]]; then
  
  # Download OpenTelemetry Collector
  wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.115.0/otelcol_0.115.0_linux_amd64.tar.gz
  tar -xzf otelcol_0.115.0_linux_amd64.tar.gz -C /tmp/
  chmod +x /tmp/otelcol
  
  # Capture cluster metadata from environment
  CLUSTER_ID=$(cat /databricks/spark/conf/cluster-id)
  CLUSTER_NAME="${CLUSTER_NAME:-unknown}"
  
  # Configure Spark to expose Prometheus metrics
  cat >> /databricks/spark/conf/spark-defaults.conf <<EOF
spark.ui.prometheus.enabled=true
spark.executor.processTreeMetrics.enabled=true
spark.metrics.conf.*.sink.prometheusServlet.class=org.apache.spark.metrics.sink.PrometheusServlet
spark.metrics.conf.*.sink.prometheusServlet.path=/metrics/prometheus
spark.metrics.conf.*.source.jvm.class=org.apache.spark.metrics.source.JvmSource
spark.metrics.appStatusSource.enabled=true
EOF
  
  # Create collector configuration
  cat > /tmp/otel-config.yaml <<EOF
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'spark-driver'
          scrape_interval: 15s
          metrics_path: '/metrics/prometheus'
          static_configs:
            - targets: ['localhost:4040']
        
        - job_name: 'spark-executors'
          scrape_interval: 15s
          metrics_path: '/metrics/executors/prometheus'
          static_configs:
            - targets: ['localhost:4040']

processors:
  resource:
    attributes:
      - key: databricks.cluster.id
        value: ${CLUSTER_ID}
        action: insert
      - key: databricks.cluster.name
        value: ${CLUSTER_NAME}
        action: insert
  
  batch:
    timeout: 10s

exporters:
  otlphttp:
    endpoint: "https://your-backend.com/v1/metrics"
    headers:
      Authorization: "Bearer YOUR_TOKEN"

service:
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: [resource, batch]
      exporters: [otlphttp]
EOF
  
  # Start collector with 30-second delay
  # (Spark UI port 4040 becomes available after SparkContext initialization)
  (sleep 30 && /tmp/otelcol --config /tmp/otel-config.yaml) &
fi
```

This script handles several critical operational concerns:

**Port availability timing:** Apache Spark's web UI on port 4040 becomes available only after SparkContext fully initializes, typically 30-60 seconds after cluster startup. The 30-second sleep prevents scrape failures during this initialization window.

**Cluster metadata enrichment:** The resource processor adds `databricks.cluster.id` and `databricks.cluster.name` attributes to all metrics, enabling filtering by cluster in backend systems and correlation with workspace-level job data.

**Dual endpoint scraping:** The collector scrapes two Prometheus endpoints on the Spark driver. The `/metrics/prometheus` endpoint exposes driver-specific metrics (DAGScheduler operations, application counters, BlockManager statistics), while `/metrics/executors/prometheus` provides aggregated executor metrics across all workers (task execution, shuffle operations, memory pressure).

**Spark Metrics Collected:**

The init script deployment captures approximately 50 internal Spark metrics that workspace APIs cannot access:

- Executor memory usage (heap, off-heap, storage, execution)
- Active, completed, and failed task counts
- Shuffle read and write volumes
- Spill to disk occurrences
- JVM garbage collection time and frequency
- Stage durations and task distribution
- RDD block status
- Streaming batch processing rates (for Structured Streaming jobs)
- Query execution plan details

These metrics provide the operational context required for troubleshooting production incidents: which executor ran out of memory, which stage caused the bottleneck, and how shuffle operations affected overall performance.

### Deployment Pattern

The two-tier architecture deploys independently:

**External Workspace Receiver:**
```yaml
# Runs on your infrastructure (VM, Kubernetes, etc.)
receivers:
  databricks:
    host: "https://your-workspace.cloud.databricks.com"
    token: "${DATABRICKS_API_TOKEN}"
    collection_interval: "60s"

exporters:
  otlphttp:
    endpoint: "https://your-backend.com/v1/metrics"

service:
  pipelines:
    metrics:
      receivers: [databricks]
      exporters: [otlphttp]
```

**Cluster-Internal Collector:**
1. Upload init script to DBFS: `dbfs:/databricks/init-scripts/otel-collector.sh`
2. Configure cluster to use init script via Databricks UI or API
3. Collector automatically deploys on every cluster startup
4. Metrics flow to the same backend as workspace receiver

Both data streams merge in your observability backend (ClickHouse, Honeycomb, Grafana, etc.), providing unified visibility across workspace orchestration and cluster execution layers.

## Why This Approach Matters for Enterprises

Many Fortune 500 enterprises are evaluating migrations away from expensive DataDog or Dynatrace contracts. For Databricks workloads, these migrations face a critical challenge: achieving observability parity with significantly lower total cost of ownership.

The two-tier OpenTelemetry architecture delivers this parity while preserving vendor neutrality:

**Cost arbitrage:** DataDog charges per host and per custom metric. A 100-node Databricks cluster running 10 jobs generates thousands of metrics across ephemeral compute resources. The resulting monthly bills frequently exceed six figures for large-scale data engineering workloads. OpenTelemetry with open-source backends like ClickHouse or commercial platforms like Honeycomb reduces these costs by 70-90% while preserving dimensional data.

**Vendor neutrality:** The workspace receiver and init script both export OTLP (OpenTelemetry Protocol), allowing you to route telemetry to any compatible backend. Switching from Honeycomb to Grafana Cloud or adding Dash0 requires only exporter configuration changes—no code modification and no data source reimplementation.

**High-cardinality data preservation:** Commercial APM vendors often drop or pre-aggregate high-cardinality dimensions to manage backend costs. OpenTelemetry with ClickHouse-based backends handles cardinality naturally through columnar storage, preserving executor IDs, task keys, and other dimensional attributes that vendors would strip away. This preserved context is essential for troubleshooting complex Spark job failures.

**Deployment flexibility:** The init script approach works universally across Databricks deployment models: AWS, Azure, GCP, classic clusters, high-concurrency clusters, and serverless SQL warehouses (workspace receiver only). You are not constrained by vendor agent compatibility matrices or platform-specific instrumentation requirements.

## Comparing OpenTelemetry to Proprietary Integrations

### DataDog Databricks Integration

DataDog's integration achieves comprehensive monitoring through similar two-tier architecture:

**Strengths:**
- Mature agent deployment patterns with extensive documentation
- Automatic correlation between workspace and cluster metrics
- Pre-built dashboards and alerting templates
- Integration with DataDog's broader APM ecosystem

**Limitations:**
- Vendor lock-in through proprietary data formats and APIs
- Per-host and per-metric pricing creates cost unpredictability
- Limited support for custom metric transformations
- Cardinality constraints enforced at agent level to manage backend costs

### Dynatrace OneAgent

Dynatrace provides arguably the most sophisticated Databricks monitoring through OneAgent deployment:

**Strengths:**
- Automatic topology discovery and dependency mapping
- Davis AI for anomaly detection and root cause analysis
- Seamless integration between workspace and cluster telemetry
- Superior UI and visualization capabilities

**Limitations:**
- Highest cost among major APM vendors
- Complete vendor lock-in with proprietary Grail backend
- Limited data export capabilities for compliance or archival needs
- Agent deployment constraints on certain Databricks runtimes

### OpenTelemetry Two-Tier Architecture

**Strengths:**
- Vendor-neutral telemetry pipeline allowing backend flexibility
- Complete control over metric cardinality and retention policies
- Significantly lower total cost of ownership
- Transparent observability into collection and processing logic
- Open-source receiver code available for security audits and customization

**Limitations:**
- Requires custom receiver development for workspace metrics
- No pre-built dashboards or alerting templates (must build or adopt community dashboards)
- Init script management adds operational complexity
- Manual correlation between workspace and cluster metrics in backend queries

For enterprises with observability expertise and engineering capacity, the OpenTelemetry approach offers superior long-term value. For organizations prioritizing rapid deployment with minimal customization, proprietary solutions provide faster time-to-value despite higher costs.

## Operational Considerations

Deploying this two-tier architecture at scale requires attention to several operational concerns:

### Init Script Management

Init scripts must be versioned, tested, and deployed consistently across cluster configurations. Databricks provides init script storage in DBFS with version control through workspace APIs. Production deployments should:

- Store init scripts in source control with proper change management
- Test script modifications on non-production clusters before rollout
- Implement rollback procedures for failed deployments
- Monitor init script execution logs for failures or configuration errors

### Collection Interval Tuning

The workspace receiver's collection interval balances metric freshness against API rate limits. Databricks enforces rate limits on REST API requests (typically 10-30 requests per second depending on workspace tier). A 60-second collection interval generating 15 API requests per scrape cycle stays well within limits while providing reasonable metric resolution for job monitoring.

Cluster-internal Prometheus scraping can operate at higher frequencies (15-30 seconds) since it queries localhost endpoints without external rate limit concerns. However, aggressive scraping increases collector resource consumption and network egress costs.

### High Availability and Failover

The workspace receiver should deploy with high availability patterns: multiple replicas behind a load balancer or active-passive failover with health checks. Init script deployments automatically achieve HA through Databricks cluster management—each cluster runs its own collector, and cluster failures take down only their associated collectors without affecting others.

### Security and Authentication

Both tiers require proper credential management:

- Workspace receiver uses Databricks personal access tokens or service principal credentials with read-only permissions
- Init scripts access Spark metrics without authentication (localhost access only)
- OTLP exporters require backend authentication tokens stored securely via environment variables or secret management systems

### Metric Cardinality Management

Databricks environments generate high-cardinality metric streams, particularly for executor IDs and task identifiers. Production deployments should implement cardinality controls:

- Filter low-value metrics at the collector level before export
- Aggregate metrics with excessive cardinality (e.g., sum task durations by stage rather than preserving per-task metrics)
- Configure backend retention policies that drop old high-cardinality data more aggressively than aggregated metrics
- Monitor backend storage growth and adjust collection strategies accordingly

## The Path Forward for Databricks Observability

The Databricks observability landscape is maturing, but significant gaps remain between marketing claims and production reality. Building comprehensive monitoring requires understanding that no single approach provides complete visibility.

**If you only need workspace-level visibility:** The custom OpenTelemetry workspace receiver provides sufficient telemetry for job scheduling, cost monitoring, and user management. This covers 80% of typical observability needs for organizations running batch ETL workloads with predictable resource consumption.

**If you need operational troubleshooting capability:** Add init script deployment for cluster-internal Spark metrics. This combination delivers the comprehensive observability required for production Spark applications where performance matters and incidents require rapid root cause analysis.

**If you want to avoid building custom receivers:** Use proprietary integrations from DataDog or Dynatrace, accepting the vendor lock-in and cost implications as trade-offs for faster deployment and pre-built dashboards.

**If you need both comprehensive observability and vendor neutrality:** Build the two-tier OpenTelemetry architecture. The upfront engineering investment pays dividends through long-term cost savings, operational flexibility, and complete control over your telemetry pipeline.

The observability ecosystem is moving toward open standards, but that movement requires engineering effort to fill gaps where vendor integrations still lead on features and ease of deployment. For Databricks specifically, the two-tier architecture represents the current state of the art for vendor-neutral comprehensive monitoring.

## References

- [Custom Databricks Receiver GitHub Repository](https://github.com/npcomplete777/databricksreceiver)
- [Apache Spark Monitoring Documentation](https://spark.apache.org/docs/latest/monitoring.html)
- [Databricks Init Scripts Documentation](https://docs.databricks.com/en/init-scripts/index.html)
- [OpenTelemetry Collector Builder Documentation](https://opentelemetry.io/docs/collector/custom-collector/)
- [Spark Prometheus Metrics](https://spark.apache.org/docs/latest/monitoring.html#metrics)
- [Databricks REST API Reference](https://docs.databricks.com/api/workspace/introduction)
