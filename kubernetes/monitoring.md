<!-- markdownlint-disable MD024 MD034 -->
# Monitoring

- [Monitoring](#monitoring)
  - [Accessing the monitoring dashboards](#accessing-the-monitoring-dashboards)
  - [Kubernetes application monitoring](#kubernetes-application-monitoring)
    - [Example ServiceMonitor](#example-servicemonitor)
    - [Example PrometheusRule](#example-prometheusrule)
    - [Using Grafana to fire alerts](#using-grafana-to-fire-alerts)
    - [Example Grafana Dashboard](#example-grafana-dashboard)
  - [AWS services monitoring](#aws-services-monitoring)
  - [Recommendations and best practices](#recommendations-and-best-practices)
    - [Prometheus labels](#prometheus-labels)
  - [Prometheus scrapers for common technologies](#prometheus-scrapers-for-common-technologies)
    - [PHP](#php)
      - [PHP-FPM](#php-fpm)
    - [Ruby](#ruby)
      - [Workers](#workers)
    - [Nginx](#nginx)
      - [Setup](#setup)
    - [RabbitMQ](#rabbitmq)
  - [Thanos](#thanos)
    - [Architecture](#architecture)
    - [How to enable Thanos in a cluster?](#how-to-enable-thanos-in-a-cluster)
    - [How to enable Prometheus remote write?](#how-to-enable-prometheus-remote-write)
    - [How to add Thanos rules?](#how-to-add-thanos-rules)

We use a [Prometheus](https://prometheus.io/), [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) and [Grafana](https://grafana.com/) stack for a complete monitoring setup of both our clusters and applications running on them.

For multi-cluster setups one can optionally use [Thanos](https://thanos.io/) to aggregate metrics from multiple Prometheus instances and store them in a single place. This allows us to have a single Grafana instance to visualize metrics from all clusters. Prometheus is used with remote write.

We make use of the CoreOS _operator_ principle: by deploying [prometheus-operator](https://github.com/coreos/prometheus-operator) we define new Kubernetes Custom Resource Definitions (CRD) for `Prometheus`, `Alertmanager`, `ServiceMonitor` and `PrometheusRule` which are responsible for deploying Prometheus and Alertmanager setups, Prometheus scrape targets and alerting rules, respectively.

As of this writing there isn't an operator setup yet for Grafana, but you can add custom dashboards dynamically via `ConfigMaps`.

- Prometheus does the _service monitoring_ and keeps time-series data.
- Alertmanager is responsible for handling alerts, based on _rules_ from Prometheus. In our case Alertmanager is responsible for making sure alerts end up in [Opsgenie](https://www.opsgenie.com/) and Slack.
- Grafana provides nice charts and dashboards of the Prometheus time-series data.

## Accessing the monitoring dashboards

Prometheus, Alertmanager and Grafana are accessible via their respective dashboards. These dashboards are exposed via Ingresses, either private or public, depending on your cluster's configuration. If the dashboards are set to be private, you'll need to be connected to the cluster's VPN in order to access them. These dashboards also require authentication, which [is setup using an Identity Provider of choice (via DEX)](authentication.md) during the initial Platform setup.

These dashboards can be reached via the following URLs (make sure to replace the use the correct cluster FQDN):

- Grafana: `https://grafana.production.eks.example.com`
- Prometheus: `https://prometheus.production.eks.example.com`
- Alertmanager: `https://alertmanager.production.eks.example.com`

Your environment-specific URLs will also be documented in your own documentation repository.

## Kubernetes application monitoring

You can also use Prometheus to monitor your application workloads and get alerts when something goes wrong. In order to do that you'll need to define your own `ServiceMonitors` and `PrometheusRules`. There are two requirementes though:

- All `ServiceMonitors` and `PrometheusRules` you define need to **have the `prometheus` label (any value will do)**.
- The **namespace** where you create your `ServiceMonitors` and `PrometheusRules` need to **have the `prometheus` label too (any value will do)**.

   ```bash
   kubectl label namespace yournamespace prometheus=true
   ```

Custom Grafana Dashboards can also be added to this same namespace by creting a `ConfigMap` with **a `grafana_dashboard` label (any value will do)**, containing the dashboard's json data. Please make sure that `"datasources"` are set to `"Prometheus"` in your dashboards!

> [!NOTE]
> even if these objects are created in a specific namespace, Prometheus can scrape metric targets in all namespaces.

It is possible to configure alternative routes and receivers for Alertmanager. This is done in your cluster definition file under the addons section. Example:

   ```yaml
   spec:
    cluster_monitoring:
      alertmanager:
        custom_routes: |
          - match:
              namespace: my-namespace
            receiver: custom-receiver
   ```

- [Upstream documentation](https://prometheus.io/docs/alerting/configuration/#route)

   ```yaml
   spec:
    cluster_monitoring:
      alertmanager:
        custom_receivers_payload:
          | # (The whole yaml block should be encrypted via KMS with the context 'k8s_stack=secrets')
          - name: custom-receiver
            webhook_configs:
              - send_resolved: true
                url: <opsgenie_api_url>
   ```

- [Upstream documentation](https://prometheus.io/docs/alerting/configuration/#receiver)

> [!NOTE]
>This configuration can be made by creating a PR to your repo (optional), and/or communicated to your Customer Lead because this needs to be rolled out to the cluster.

### Example ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    prometheus: application
  name: myapplication-php-fpm-exporter
  namespace: mynamespace
spec:
  endpoints:
    - targetPort: 8080
      interval: 30s
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: application
      component: php
```

You can find more examples in the official [Prometheus Operator documentation](https://github.com/coreos/prometheus-operator/tree/master/Documentation/user-guides) or get inspired by the ones already deployed in the cluster:

```bash
kubectl get servicemonitors --all-namespaces
```

### Example PrometheusRule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: application
  name: myapplication-alert-rules
  namespace: mynamespace
spec:
  groups:
  - name: myapplication.rules
    rules:
    - alert: MyApplicationPhpFpmDown
      expr: phpfpm_up{job="application"} == 0
      for: 10m
      labels:
        severity: critical
        namespace: namespace
      annotations:
        description: '{{`{{ $value }}`}}% of {{`{{ $labels.job }}`}} PHP-FPM are down!'
        summary: PHP-FPM down
        runbook_url: 'https://github.com/myorg/myapplication/tree/master/runbook.md#alert-name-myapplicationphpfpmdown'
```

You can find more examples in the official [Prometheus Operator documentation](https://github.com/coreos/prometheus-operator/tree/master/Documentation/user-guides) or get inspired by the ones already deployed in the cluster:

```bash
kubectl get prometheusrules --all-namespaces
```

> [!NOTE]
> We use the `namespace` label in the alerts, to distinguish between infrastructure and application alerts, and distribute them to the appropriate receivers. If the namespace label is set to any of the namespaces we are responsible for (e.g. infrastructure, cert-manager, keda, istio-system, ...), or if the alert doesn't have a namespace label (a sort of catch-all), we consider them infrastructure, and are routed to our on-call system and Slack channels. Otherwise the alerts are considered application alerts and routed to the `#devops_<customername>_alerts` Slack channel (and any other receivers that you define in the future). This label can be "hardcoded" as an alert label in the alert definition, or exposed from the alert expression. This is important since some alerts span multiple namespaces and it's desirable to get the namespace from which the alert originated.
> [!TIP]
> we highly recommend including a `runbook_url` annotation to all alerts so the engineer that handles those alerts has all the needed information and can troubleshoot issues faster.

### Using Grafana to fire alerts

You leverage the Grafana alerting system to fire alerts based on the data you have in Prometheus. This is done by creating a Grafana dashboard (or setting up [Alert Rules](https://grafana.com/docs/grafana/latest/alerting/fundamentals/alert-rules/)) and setting up alerts on it.

By default Skyscrapers managed Grafana has a built in integration with Alertmanager as a [Contact Point](https://grafana.com/docs/grafana/latest/alerting/fundamentals/notifications/contact-points/).
Make sure to use the `Main Alertmanager` as the contact point for your alerts.
To make sure that Grafana persistently stores the alerts, you need to make sure the Grafana workload has persistent storage. You can do this by setting the `spec.cluster_monitoring.grafana.persistence.enabled` to `true` in your cluster definition file.

This means that you can set up alerts in Grafana and they will be sent to Alertmanager, which will then route them to the correct receiver.

### Example Grafana Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    grafana_dashboard: application
  name: grafana-dashboard-myapplication
  namespace: mynamespace
data:
  grafana-dashboard-myapplication.json: |
{ <Grafana dashboard json> }
```

You can get inspired by some of the dashboards already deployed in the cluster:

```bash
kubectl get configmaps -l grafana_dashboard --all-namespaces
```

## AWS services monitoring

AWS services can also be monitored via Prometheus and Alertmanager, like the rest of the cluster. To do so we use the [Prometheus cloudwatch-exporter](https://github.com/prometheus/cloudwatch_exporter), which imports CloudWatch metrics into Prometheus. From there we can build alerts that trigger when some conditions happen.

The cloudwatch-exporter is not deployed by default as a base component of the reference solution, as it's highly dependent on the customer needs.

We provide pre-made Helm charts for some AWS resources, like [RDS](https://github.com/skyscrapers/charts/tree/master/rds-monitoring), [Redshift](https://github.com/skyscrapers/charts/tree/master/redshift-monitoring) and [Elasticsearch](https://github.com/skyscrapers/charts/tree/master/elasticsearch-monitoring), which deploy the cloudwatch-exporter and some predefined alerts, but additional cloudwatch-exporters can be deployed to import any other metrics needed. Be aware that exporting data from Cloudwatch is quite costly, and every additional exported metric requires additional API calls, so make sure you only export the metrics you'll use.

Normally, you'll deploy a different cloudwatch-exporter for each AWS service that you want to monitor, as each of them will probably require different period configurations.

You'll also need an IAM role for the cloudwatch-exporter with the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricStatistics",
        "tag:GetResources"
      ],
      "Resource": "*"
    }
  ]
}
```

> [!NOTE]
> A single role can be used for all cloudwatch exporters deployed on the same cluster.

## Recommendations and best practices

### Prometheus labels

As the [Prometheus documentation](https://prometheus.io/docs/practices/naming/#labels) states:
> Use labels to differentiate the characteristics of the thing that is being measured:
>
> - api_http_requests_total - differentiate request types: type="create|update|delete"
> - api_request_duration_seconds - differentiate request stages: stage="extract|transform|load"
<!--  -->
> [!CAUTION]
> Remember that every unique combination of key-value label pairs represents a new time series, which can dramatically increase the amount of data stored. Do not use labels to store dimensions with high cardinality (many different label values), such as user IDs, email addresses, or other unbounded sets of values.

Not following that advise can cause the whole Prometheus setup to become unstable, go out of memory and eventually cause collateral damage on the node that's running on. A good example can be found in [this Github issue](https://github.com/prometheus/client_golang/issues/491).

This is how Prometheus would perform with "controlled" metric labels:

![prometheus-healthy](./images/prometheus-healthy.png)

Vs. a Prometheus with "uncontrolled" metric labels:

![prometheus-unhealthy](./images/prometheus-unhealthy.png)

## Prometheus scrapers for common technologies

Here's a list of Prometheus scrapers already available for common frameworks.

### PHP

Prometheus has a [native client library](https://github.com/Jimdo/prometheus_client_php) for PHP. This is easy to implement inside your application, but by default doesn't expose standard metrics. This is ideal if you want to expose your own metrics (can also be business metrics).

#### PHP-FPM

If you want to get PHP-FPM metrics, we recommend using [this exporter](https://github.com/hipages/php-fpm_exporter). It's being actively maintained and the documentation is reasonably good. You have to setup the exporter as a sidecar container in your pods/task-definition, then it'll access the PHP-FPM socket to read statistics and expose them as Prometheus metrics.

You first need to expose the metrics in PHP-FPM. You can do this by adding the following config to your PHP-FPM image.

```ini
pm.status_path = /status
```

Then you'll need to add the `php-fpm-exporter` as a sidecar container to your pod/task-definition.

Here is an example for **k8s**:

```yaml
- name: {{ template "app.fullname" . }}-fpm-exporter
  image: hipages/php-fpm_exporter:2.2.0
  env:
    - name: PHP_FPM_SCRAPE_URI
      value: "tcp://127.0.0.1:{{ .Values.app.port }}/status"
  ports:
    - name: prom
      containerPort: 9253
      protocol: TCP
  livenessProbe:
    tcpSocket:
      port: prom
    initialDelaySeconds: 10
    periodSeconds: 5
  readinessProbe:
    tcpSocket:
      port: prom
    initialDelaySeconds: 10
    timeoutSeconds: 5
  resources:
    limits:
      cpu: 30m
      memory: 32Mi
    requests:
      cpu: 10m
      memory: 10Mi
```

> [!NOTE]
> You'll need to adjust `{{ template "app.fullname" . }}` and `{{ .Values.app.port }}` to the correct helm variables. The first one represents the app name we want to monitor. The second is the php-fpm port of the application.*

### Ruby

Prometheus has a [native client library](https://github.com/prometheus/client_ruby) for Ruby. This is a really good library when you run your ruby application as single process. Unfortunately a lot of applications use Unicorn or another multi-process integration. When using a multi-process Ruby you can best use the [fork](https://gitlab.com/gitlab-org/prometheus-client-mmap) of GitLab.

You have to integrate this library in your application and expose it as an endpoint. Once that is done, you can add a `ServiceMonitor` to scrape it.

#### Workers

Workers by default don't expose a webserver to scrape from. This will have to change and every worker will need to expose a simple webserver so that Prometheus can scrape its metrics.

It is really discouraged to use Pushgateway for this. For more info why this is discouraged, see the [Pushgateway documentation](https://prometheus.io/docs/practices/pushing/#should-i-be-using-the-pushgateway).

### Nginx

Nginx has an exporter through the [Nginx VTS module](https://github.com/vozlt/nginx-module-vts). We [forked](https://github.com/skyscrapers/docker-images/tree/master/nginx) the official Docker Nginx image and added the VTS module to nginx. We had to fork it because the VTS module needs to be compiled with Nginx.

#### Setup

1. Use our [Nginx](https://hub.docker.com/r/skyscrapers/nginx/) Docker image instead of the upstream Nginx.
2. Add the [exporter](https://github.com/hnlq715/nginx-vts-exporter) as sidecar to the pod/task-definition.

Example for **k8s**:

```yaml
- name: {{ template "app.fullname" . }}-exporter
  image: "sophos/nginx-vts-exporter:v0.10.0"
  env:
    - name: NGINX_STATUS
      value: http://localhost:9999/status/format/json
  ports:
    - name: prom
      containerPort: 9913
```

> [!NOTE]
> You will need to adjust `{{ template "app.fullname" . }}` to the correct helm variables. It represents the app name you want to monitor.

### RabbitMQ

Starting from version 3.8.0, RabbitMQ ships with built-in Prometheus & Grafana support. The Prometheus metric collector needs to be enabled via the `rabbitmq_prometheus` plugin. Head to the [official documentation](https://www.rabbitmq.com/prometheus.html#overview-prometheus) to know more on how to enable and use it.

Once the `rabbitmq_prometheus` plugin is enabled, the metrics port needs to be exposed in the RabbitMQ Pods and Service. RabbitMQ uses TCP port `15692` by default.

Then a `ServiceMonitor` is needed to instruct Prometheus to scrape the RabbitMQ service. Follow the [instructions above](#example-servicemonitor) to set up the correct `ServiceMonitor`.

At this point the RabbitMQ metrics should already be available in Prometheus. We can also deploy a RabbitMQ overview dashboard in Grafana, which displays detailed graphs and metrics from the data collected in Prometheus. Reach out to your Customer Lead in case you'd be interested in such dashboard.

## Thanos

### Architecture

We opt for this architecture: Deployment via Receive in order to scale out or integrate with other remote write-compatible sources, so that we can use the Prometheus remote write feature to send metrics to Thanos.

![image:thanos](./images/Thanos_arch.png)

[For a more in depth explanation of all the components.](https://thanos.io/tip/thanos/quick-tutorial.md/)

### How to enable Thanos in a cluster?

A Skyscrapers engineer can help you to enable Thanos and/or you can update your cluster definition file, through pull request:

```yaml
spec:
  thanos:
    enabled: true
    receive:
      tenants:
        - <name of leaf cluster>
      pv_size: 100Gi
      tsdb_retention: 6h
    retention_1h: 30d
    retention_5m: 14d
    retention_raw: 14d
```

The receive pv_size and tsdb_retention are used to configure the thanos receive component. The tsdb retention is the amount of time that the data will be stored in the receive component. The retention_1h, retention_5m and retention_raw are used to configure the thanos store component. The retention_1h is the amount of time that the data will be stored in the store component for 1h resolution. The retention_5m is the amount of time that the data will be stored in the store component for 5m resolution. The retention_raw is the amount of time that the data will be stored in the store component for raw resolution.

### How to enable Prometheus remote write?

A Skyscrapers engineer can help you to enable Prometheus remote write and/or you can update your cluster definition file, through pull request:

```yaml
  cluster_monitoring:
    prometheus:
      remote_write:
        enabled: true
        name: thanos
        url: https://receive.thanos.<FQDN of Thanos cluster>/api/v1/receive
```

### How to add Thanos rules?

1. Create a yaml file with the rules you want to add. In the example we'll name it thanos-rules.yml. It will be reused in the next step.
NOTE: The file needs to end with *.yml otherwise it will not be picked up by the thanos rules.

   ```yaml
   groups:
     - name: test-thanos-alertmanager
       rules:
         - alert: TestAlertForThanos
           expr: vector(1)
           labels:
             severity: warning
           annotations:
             summary: Test alert
             description: This is a test alert to see if alert manager is working properly with Thanos
   ```

2. Overwrite the existing thanos-ruler-configmap in the infrastructure namespace with the new yaml file.

   ```bash
   kubectl create configmap thanos-ruler-config --from-file=thanos-rules.yml -n infrastructure --dry-run=client -o yaml | kubectl replace -f -
   ```
