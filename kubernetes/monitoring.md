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
kubectl get configmaps -l grafana_dashboard=grafana_dashboard=cluster-monitoring --all-namespaces
```
