# Kubernetes logging

## Grafana Loki

By default we provide [Grafana Loki](https://grafana.com/oss/loki) as a solution for centralized access to your logs.

> Unlike other logging systems, Loki is built around the idea of only indexing metadata about your logs: labels (just like Prometheus labels). Log data itself is then compressed and stored in chunks in object stores such as S3 or GCS, or even locally on the filesystem. A small index and highly compressed chunks simplifies the operation and significantly lowers the cost of Loki.

You can access this through your Grafana dashboard by clicking `Explore` in the sidebar and selecting `Loki` as datasource.

![Grafana Loki](images/grafana_loki.png "Grafana Loki")

Loki comes with it's own query language, LogQL, for which you can find the documentation here: <https://github.com/grafana/loki/blob/master/docs/logql.md>.

**Note:** the default limit of log lines in Grafana is 1000. This is the default limit and is in place to keep the Grafana resource footprint low. This is however a parameter and can be adjusted on a per-customer basis case.

Because Loki is integrated into Grafana, you can also add log panels to your application-specific dashboards in Grafana.

### Log sources

We ship the following logs to Loki

- **Application logs**: Logs that are picked up by Promtail are the same that are available through `kubectl logs`, which are the `stdout` and `stderr` streams of all containers running in a Pod. In other words, everything that's dumped in `stdout` and `stderr` of every container will be available in Loki.
- **Kubernetes events**: Kubernetes events (`kubectl get events -n <namespace>`) can be queried in Loki via `{app="eventrouter", namespace="<mynamespace>"}`
- **Kubernetes Node logs**: Kubelet logs can be queried in Loki via `{unit="kubelet.service",hostname="<node>"}`

### Architecture

Our Loki stack uses the following components:

- [Grafana Loki](https://github.com/grafana/loki/blob/master/docs/overview/README.md)
- [Promtail](https://github.com/grafana/loki/blob/master/docs/clients/promtail/README.md) which ships the container logs to Loki. Promtail is deployed as K8s Daemonset.
- [AWS S3](https://aws.amazon.com/s3/) used for storing log chunk data
- [AWS DynamoDB](https://aws.amazon.com/dynamodb/) used for indexing the log chunks
  - Tables are automatically rotated

### Metrics and alerts from Loki log contents in Grafana

It is possible to create Grafana panels based on log metrics from Loki, to do so you need to use the `PromLoki` datasource in your Grafana and use your [LogQL](https://github.com/grafana/loki/blob/master/docs/logql.md) query in the query field, as you would normally do with Prometheus metrics.

In the same manner, you can also create alerts based on log metrics using the `Alert` tab in the Grafana panels you created using the `PromLoki` datasource. Note that you need to first create a [notification channel](https://grafana.com/docs/grafana/latest/alerting/notifications/) in Grafana, so alerts can reach the desired target.

Unfortunately the notification channels needed for the alerts can only be set via the web UI, which means that if you want to use alerts, we'll need to enable persistence in your Grafana setup.

This is a short video from Grafana on how to create alerts based on log metrics: [https://www.youtube.com/watch?v=GdgX46KwKqo](https://www.youtube.com/watch?v=GdgX46KwKqo).

**Note** that Grafana doesn't run in HA mode at the moment, so alerts can't be considered 100% reliable. Although we're going to be notified if for some reason Grafana is unavailable.
