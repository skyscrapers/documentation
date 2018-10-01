# Kubernetes monitoring

We use a [Prometheus](https://prometheus.io/), [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) and [Grafana](https://grafana.com/) stack for a complete monitoring setup of both the Kubernetes cluster itself and applications running on it.

- Prometheus does the _service monitoring_ and keeps time-series data. There's one for the cluster monitoring and another one for monitoring your applications.
- Alertmanager is responsible for handling alerts, based on _Rules_ from Prometheus. In our case Alertmanager is responsible for making sure alerts end up in [Opsgenie](https://www.opsgenie.com/) and Slack.
- Grafana provides nice charts and dashboards of the Prometheus time-series data.

We make use of the CoreOS _operator_ principle: by deploying [prometheus-operator](https://github.com/coreos/prometheus-operator) we define new Kubernetes Custom Resource Definitions (CRD) for `Prometheus`, `Alertmanager`, `ServiceMonitor` and `PrometheusRule` which are responsible for deploying Prometheus and Alertmanager setups, Prometheus scrape targets and alerting rules, respectively. As of this writing there isn't an operator setup yet for Grafana.

# Kubernetes application monitoring

As mentioned before, for application monitoring we deploy a separate [Prometheus](https://prometheus.io/) setup, but we reuse the Alertmanager and Grafana from the cluster monitoring.

## Custom config

### ServiceMonitors

You can add extra [Servicemonitors](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#servicemonitor) without having to modify your values file. An example can be found [here](https://github.com/coreos/prometheus-operator/blob/master/example/user-guides/getting-started/example-app-service-monitor.yaml). By doing this your `ServiceMonitors` can be part of your application deploy. One drawback of using this approach is that you are required to deploy your `ServiceMonitors` in the same namespace as the Prometheus that has to pick them up.
Prometheus itself is able to scrape targets in different namespaces.

An example of `ServiceMonitor` would be:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: prometheus
    prometheus: application-monitor
  name: myapplication-php-fpm
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

### Alerting Rules

You can add extra Rules without having to modify your values. You can add an extra `PrometheusRule` objects with rules, the only requirement is that your `PrometheusRule` has the following labels.

```yaml
app: prometheus
prometheus: client-monitor
```

The same limitation as with `ServiceMonitors` is applied here as well. You need to make sure your `PrometheusRule` is deployed in the same namespace as the Prometheus that has to pick up those rules.

An example of an alerting rule would be:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    app: prometheus
    prometheus: client-monitor
  name: myapplication-rules
  namespace: application-monitoring
spec:
  groups:
  - name: application.rules
    rules:
    - alert: ApplicationPhpFpmDown
      expr: phpfpm_up{job="application"} == 0
      for: 10m
      labels:
        severity: warning
        client: MySuperDuperCompanyName
      annotations:
        description: '{{`{{ $value }}`}}% of {{`{{ $labels.job }}`}} PHP-FPM are down.'
        summary: PHP-FPM down
```
