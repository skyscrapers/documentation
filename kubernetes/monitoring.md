# Kubernetes monitoring

We use Prometheus for monitoring our Kubernetes clusters.
The documentation for Prometheus-specific topics can be found [here](../tools/prometheus.md).

We make use of the CoreOS _operator_ principle: by deploying [prometheus-operator](https://github.com/coreos/prometheus-operator) we define new Kubernetes Custom Resource Definitions (CRD) for `Prometheus`, `Alertmanager`, `ServiceMonitor` and `PrometheusRule` which are responsible for deploying Prometheus and Alertmanager setups, Prometheus scrape targets and alerting rules, respectively.

As of this writing there isn't an operator setup yet for Grafana, but you can add custom dashboards dynamically via `ConfigMaps`.

## Kubernetes application monitoring

You can also use Prometheus to monitor your application workloads and get alerts when something goes wrong. In order to do that you'll need to define your own `ServiceMonitors` and `PrometheusRules`. There are two requirementes though:

- All `ServiceMonitors` and `PrometheusRules` you define need to have the `prometheus` label (any value will do).
- The namespace where you create your `ServiceMonitors` and `PrometheusRules` need to have the `prometheus` label too (any value will do).

   ```bash
   kubectl label namespace yournamespace prometheus=true
   ```

Custom Grafana Dashboards can also be added to this same namespace by creting a `ConfigMap` with a `grafana_dashboard` label (any value will do), containing the dashboard's json data. Please make sure that `"datasources"` are set to `"Prometheus"` in your dashboards!

*Note: even if these objects are created in a specific namespace, Prometheus can scrape metric targets in all namespaces.*

It is possible to configure alternative routes and receivers for Alertmanager. This is done in your cluster definition file under the addons section. Example:

   ```yaml
   cluster_monitoring_custom_alertmanager_routes:
     - match:
         namespace: my-namespace
       receiver: custom-receiver
   ```

- [Upstream documentation](https://prometheus.io/docs/alerting/configuration/#route)

   ```yaml
   # (The whole yaml block should be encrypted via KMS with the context 'k8s_stack=secrets')
   cluster_monitoring_custom_alertmanager_receivers_payload:
     - name: custom-receiver
       webhook_configs:
         - send_resolved: true
           url: <opsgenie_api_url>
   ```

- [Upstream documentation](https://prometheus.io/docs/alerting/configuration/#receiver)

*Note: This configuration can be made by creating a PR to your repo (optional), and/or communicated to your lead engineer because this needs to be rolled out to the cluster.*

### Example ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    prometheus: application-monitor
  name: myapplication-php-fpm-exporter
  namespace: application-monitoring
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
    prometheus: application-monitor
  name: myapplication-alert-rules
  namespace: application-monitoring
spec:
  groups:
  - name: myapplication.rules
    rules:
    - alert: MyApplicationPhpFpmDown
      expr: phpfpm_up{job="application"} == 0
      for: 10m
      labels:
        severity: critical
        client: MyCompanyName
      annotations:
        description: '{{`{{ $value }}`}}% of {{`{{ $labels.job }}`}} PHP-FPM are down!'
        summary: PHP-FPM down
        runbook_url: 'https://github.com/myorg/myapplication/tree/master/runbook.md#alert-name-myapplicationphpfpmdown'
```

You can find more examples in the official [Prometheus Operator documentation](https://github.com/coreos/prometheus-operator/tree/master/Documentation/user-guides) or get inspired by the ones already deployed in the cluster:

```bash
kubectl get prometheusrules --all-namespaces
```

**Note:** we highly recommend including a `runbook_url` annotation to all alerts so the engineer that handles those alerts has all the needed information and can troubleshoot issues faster.

### Example Grafana Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    grafana_dashboard: application-monitor
  name: grafana-dashboard-myapplication
  namespace: application-monitoring
data:
  grafana-dashboard-myapplication.json: |
{ <Grafana dashboard json> }
```

You can get inspired by some of the dashboards already deployed in the cluster:

```bash
kubectl get configmaps -l grafana_dashboard=cluster-monitoring --all-namespaces
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

_Note that a single role can be used for all cloudwatch exporters deployed on the same cluster_
