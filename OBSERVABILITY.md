# Observability — Session 20, Task 2

Bibek Jyoti Charah — 24bcs10112 (GitHub: SammyUrfen)

Each section ties back to what I ran in [README.md](README.md).

## Monitoring vs observability

Monitoring answers questions you planned for: "is the app up?", "is memory over 50 MiB?". You write
the check before the failure happens.

Observability is the ability to answer questions you did not plan for, from the data the system
already emits. When a new kind of failure appears, you can find the cause without shipping new code.

Monitoring is one use of observability data. Good observability makes monitoring easy to add.

## The three pillars

| Pillar | What it is | Shape of the data | Good for | Weak at | In this homework |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Metrics | Numbers measured over time, with labels | `up{job="demo-app",pod="..."} 1` every 15 s | Trends, dashboards, alerts. Cheap to store. | The detail of one single request | Prometheus scraped 10 jobs. I queried CPU, memory, `up`, replicas and restarts. |
| Logs | Text events, one line per thing that happened | `level=INFO msg="Starting rule manager..."` | The exact error and its context | Totals and trends. Expensive at volume. | `kubectl logs` on Prometheus and the Argo CD controller. |
| Traces | The path of one request through many services, as timed spans | A tree: gateway 120 ms → orders 80 ms → DB 60 ms | Finding which hop in a chain is slow or failing | Needs code instrumentation in every service | Not run. The demo is one service, so a trace has only one span. |

How they work together: a metric alert says *something* is wrong. A trace shows *where* in the
request path. A log line at that place shows *why*.

## Why observability is required

- **Distributed systems fail in partial ways.** One of three replicas can be slow while the others
  are fine. An average hides it. Per-pod metrics show it.
- **Containers are short-lived.** A crashed pod and its local logs are gone after a restart. The data
  must leave the pod and go to a central store.
- **Faster recovery.** Mean time to detect and mean time to repair drop when the data is already
  collected. In my run, Prometheus showed `DemoAppDown` firing about 60 s after the app went to zero
  replicas, with no person watching.
- **Capacity and cost.** The memory query showed Grafana at about 535 MiB, which is more than all
  other monitoring pods together. Without numbers I would have guessed wrong.
- **Proof for changes.** After a deploy you can compare before and after, not guess.

## Common tools

| Need | Open source | Hosted / commercial |
| :--- | :--- | :--- |
| Metrics store and query | **Prometheus** (used here), Thanos, Mimir, VictoriaMetrics | Datadog, New Relic, CloudWatch |
| Dashboards | **Grafana** (used here) | Datadog, Grafana Cloud |
| Alert routing | **Alertmanager** (used here) | PagerDuty, Opsgenie |
| Logs | Loki, Elasticsearch / OpenSearch + Kibana, Fluent Bit / Fluentd (shippers) | Splunk, CloudWatch Logs |
| Traces | Jaeger, Tempo, Zipkin | Datadog APM, Honeycomb, X-Ray |
| Instrumentation standard | **OpenTelemetry** (SDKs + Collector for all three pillars) | — |

## Kubernetes observability

Kubernetes gives several data sources. kube-prometheus-stack wires most of them into Prometheus.

| Source | What it tells you | Used here |
| :--- | :--- | :--- |
| kubelet / cAdvisor | CPU and memory per container and pod | `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes` |
| kube-state-metrics | The state of API objects: replicas, restarts, pod phase | `kube_deployment_status_replicas_available`, `kube_pod_container_status_restarts_total` |
| node-exporter | Node CPU, memory, disk, network | `node_memory_*` (see Findings: on minikube's Docker driver it reads the host) |
| API server, CoreDNS | Control plane health | Scraped, `up == 1` |
| Application `/metrics` | Business and request metrics | The demo app, found by a `ServiceMonitor` |
| Probes | Readiness takes a pod out of the Service. Liveness restarts it. | HTTP probes on the demo app |
| Events | Short-lived records of what the cluster did | `kubectl get events` on the Argo CD Application |
| Container logs | stdout/stderr per container, kept on the node | `kubectl logs` |

The Prometheus Operator adds custom resources so monitoring config lives next to the app:
`ServiceMonitor` (what to scrape) and `PrometheusRule` (what to alert on). I applied both with
`kubectl apply`, and Prometheus loaded them with no restart.

### Where a log stack fits

`kubectl logs` reads from the node, one pod at a time, and only while the pod exists. A log stack
fixes that. With Loki, a DaemonSet agent (Promtail or Grafana Alloy) tails
`/var/log/pods/*` on every node, adds the pod labels, and pushes the lines to Loki. Grafana then
queries logs with LogQL using the same labels as the Prometheus metrics. That lets you jump from a
spike on a graph to the log lines of the same pod and time. I did not install Loki. See the README
for why.
