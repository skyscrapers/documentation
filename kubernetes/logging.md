# Kubernetes logging

- [Kubernetes logging](#kubernetes-logging)
  - [Grafana Loki](#grafana-loki)
    - [Log sources](#log-sources)
    - [LogQL examples](#logql-examples)
    - [Architecture](#architecture)
    - [Metrics and alerts from Loki log contents in Grafana](#metrics-and-alerts-from-loki-log-contents-in-grafana)
  - [Other Log systems](#other-log-systems)

## Grafana Loki

Out of the box we provide [Grafana Loki](https://grafana.com/oss/loki) as a solution for centralized access to your logs.

> Unlike other logging systems, Loki is built around the idea of only indexing metadata about your logs: labels (just like Prometheus labels). Log data itself is then compressed and stored in chunks in object stores such as S3 or GCS, or even locally on the filesystem. A small index and highly compressed chunks simplifies the operation and significantly lowers the cost of Loki.

You can access this through your Grafana dashboard by clicking `Explore` in the sidebar and selecting `Loki` as datasource.

![Grafana Loki](images/grafana_loki.png "Grafana Loki")

Loki comes with it's own query language, LogQL, for which you can find the documentation here: <https://grafana.com/docs/loki/latest/logql/>.

**Note:** the default limit of log lines which Grafana will display is 1000 (with a max query size of 5000 records). This is the default limit and is in place to keep the Grafana resource footprint low. This is however a parameter and can be adjusted on a per-customer basis case.

Because Loki is integrated into Grafana, you can also add log panels to your application-specific dashboards in Grafana.

### Log sources

We ship the following logs to Loki:

- **Application logs** (optional, default): The same log that are available through `kubectl logs`, which are the `stdout` and `stderr` streams of all containers running in a Pod. Enriched with Kubernetes metadata to query on, eg. `pod_name`, `namespace_name`, `container_name`, `host`, ... and any Kubernetes labels you've set on your workloads. Example query `{job="kube",namespace_name="production",app_kubernetes_io_name="myapp"}`
- **Infrastructure and system logs** (always): Same as above, but for the system namespaces.
- **Kubernetes events** (always): Kubernetes events (`kubectl get events -n <namespace>`) can be queried in Loki with eg. `{job="kube",app="eventrouter"} | json | event_metadata_namespace="<namespace>"`
- **Kubelet logs** (always): Kubelet logs can be queried in Loki via `{job="kubelet",host="<node>"}`

### LogQL examples

*Make sure to check out the official documentation at <https://grafana.com/docs/loki/latest/logql/>.*

```logql
# Get logs from the production namespaces of myapp
{job="kube",namespace_name="production",app_kubernetes_io_name="myapp"}

# Get logs from the production namespaces of myapp, if you're using older style labels
{job="kube",namespace_name="production",app="myapp"}

# Get logs from the production namespaces of myapp, parse the log's json fields, and search where http_response is 404 
{job="kube",namespace_name="production",app_kubernetes_io_name="myapp"} | json | http_response="404"

# Get logs from the production namespaces of myapp, parse the log's json fields, and show only the values of field http_message
{job="kube",namespace_name="production",app_kubernetes_io_name="myapp"} | json | line_format "{{ .http_message }}"
```

### Architecture

Our Loki stack uses the following components:

- [Grafana Loki](https://grafana.com/docs/loki/latest/)
- [Fluent Bit](https://docs.fluentbit.io/manual/) which ships the container logs to Loki. Fluent Bit is deployed as K8s Daemonset.
- [AWS S3](https://aws.amazon.com/s3/) used for storing log chunk data
- [AWS DynamoDB](https://aws.amazon.com/dynamodb/) used for indexing the log chunks
  - Tables are automatically rotated

### Metrics and alerts from Loki log contents in Grafana

It is possible to create Grafana panels based on log metrics from Loki, to do so you need to use the `PromLoki` datasource in your Grafana and use your [LogQL](https://github.com/grafana/loki/blob/master/docs/logql.md) query in the query field, as you would normally do with Prometheus metrics.

In the same manner, you can also create alerts based on log metrics using the `Alert` tab in the Grafana panels you created using the `PromLoki` datasource. Note that you need to first create a [notification channel](https://grafana.com/docs/grafana/latest/alerting/notifications/) in Grafana, so alerts can reach the desired target.

Unfortunately the notification channels needed for the alerts can only be set via the web UI, which means that if you want to use alerts, we'll need to enable persistence in your Grafana setup.

This is a short video from Grafana on how to create alerts based on log metrics: [https://www.youtube.com/watch?v=GdgX46KwKqo](https://www.youtube.com/watch?v=GdgX46KwKqo).

**Note** that Grafana doesn't run in HA mode at the moment, so alerts can't be considered 100% reliable. Although we're going to be notified if for some reason Grafana is unavailable.

## Other Log systems

By standardizing on Fluent Bit for log shipping, we can use the same system for sending logs to other sources, like: S3, Elasticsearch / Opensearch, [Logz.io](https://logz.io/), ... Get in touch if you want to use any of those systems.
