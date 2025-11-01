# Observability Blueprint for Java Spring Boot Backend

## Overview
Observability is the ability to understand the internal state of a system by examining its outputs. For a Spring Boot backend, this means instrumenting the application so that logs, metrics and traces provide enough information to monitor, debug and tune performance. This blueprint describes how to implement logging, metrics and tracing in a unified way, how to integrate with the ELK stack, and how to capture relevant metrics for Kafka and Redis when they are used as infrastructure components.

## Logging
- **Structured logging:** Use Logback or Log4J2 with a JSON layout so that each log entry contains a timestamp, level, logger name, message and key-value fields (such as trace and span IDs). Structured logs are easier to parse and query in Elasticsearch. Enable asynchronous appenders to reduce I/O overhead.
- **Context propagation:** Use Spring Cloud Sleuth or Micrometer Tracing to automatically add `traceId` and `spanId` to MDC so that every log line can be correlated with the corresponding trace. This correlation makes it possible to jump from a problematic trace to the exact log entries.
- **Log levels:** Configure sensible defaults (INFO for framework packages, DEBUG for your application packages when debugging). Use environment variables to adjust verbosity without code changes.
- **Log shipping:** Avoid writing logs to local disk in containerized deployments. Configure a Logstash or Filebeat sidecar/agent to collect logs from stdout and ship them to a central Logstash instance. Logstash can parse the JSON log lines, enrich them (e.g., add host and environment metadata) and send them to Elasticsearch.

## Metrics
- **Micrometer and Actuator:** Include `spring-boot-starter-actuator` and `micrometer-registry-prometheus` or another registry. Actuator exposes built‑in metrics such as JVM memory usage, CPU load, request counts and HTTP latency via `/actuator/metrics` and vendor-specific endpoints (e.g., `/actuator/prometheus`). Micrometer provides a vendor‑neutral API for creating counters, gauges, timers and distribution summaries.
- **Custom metrics:** Instrument important code paths (e.g., external API calls, database operations) using `@Timed` or programmatic timers. Use tags (low‑cardinality key–value pairs) to distinguish between operations, endpoints, tenant IDs, etc.
- **Kafka metrics:** When using Spring for Apache Kafka, enable Micrometer metrics for producers and consumers. Monitor broker throughput (BytesInPerSec, BytesOutPerSec, MessagesInPerSec), producer request latency, consumer lag (offset lag per partition), under‑replicated partitions, active controller count and JVM memory usage. These metrics help detect throughput bottlenecks, replication issues and lag.
- **Redis metrics:** When using Spring Data Redis (Lettuce driver), enable Micrometer integration. Monitor command execution time, operations per second, cache hit ratio, used memory vs. max memory, memory fragmentation ratio, replication lag, keyspace hits/misses, expired and evicted keys, slow log entries and connection counts. These metrics reveal cache efficiency and resource pressure.
- **Metrics export:** Choose a backend to store metrics. Elastic supports ingesting metrics through its Spring Boot integration, Prometheus can scrape the `/actuator/prometheus` endpoint, and other vendors (Datadog, New Relic, etc.) are supported by Micrometer. Select the system that matches your organisation’s observability stack.

## Tracing
- **Distributed tracing:** Use Micrometer Tracing (formerly part of Spring Cloud Sleuth) to create spans for inbound and outbound requests. Each HTTP request is assigned a trace ID and spans are created for client/server calls, messaging operations and database interactions. Use the OpenTelemetry bridge so that instrumentation is vendor‑neutral.
- **Propagation:** Ensure that the W3C trace context is propagated over HTTP headers (`traceparent`, `tracestate`) and messaging headers (Kafka headers). This allows traces to be correlated across services written in different languages.
- **Exporters:** Configure a tracing exporter such as Zipkin, Jaeger, Tempo or Elastic APM. Exporters collect spans, assemble them into traces and provide a UI to analyse latency and dependencies. Choose a sampling strategy (e.g., 0.1% in production) to balance cost and visibility.

## ELK/Logstash Integration
- **Elasticsearch:** Use Elasticsearch as the central datastore for logs and (optionally) metrics. It provides fast full‑text search and aggregation capabilities.
- **Logstash pipeline:** Configure Logstash to accept logs via Beats or TCP, parse the JSON log lines and index them into Elasticsearch. Use filters to rename fields, drop verbose keys and extract timestamps.
- **Kibana:** Create dashboards that display log volume over time, top error messages, slow requests and key metrics such as HTTP response times. Kibana allows interactive drill‑down and correlations between logs and metrics. A sample dashboard JSON is provided in `dashboard.json`.

## Kafka and Redis Integration
- **Instrumentation:** Ensure that Kafka producers, consumers and streams are instrumented through the Spring Observability support. Micrometer automatically creates timers and counters for each operation. Similarly, Spring Data Redis instruments Lettuce commands.
- **Alerting:** Configure alert rules in your metrics backend based on Kafka and Redis metrics. Examples include alerts on high consumer lag, under‑replicated partitions, high command latency or low cache hit ratio.
- **Trade‑offs:** Instrumentation adds slight overhead; choose meaningful sampling rates and metric resolutions. Centralized logging and metrics ingestion may incur storage and cost overhead; configure retention policies and sampling accordingly.
