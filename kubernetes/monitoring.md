# Kubernetes monitoring

We use a [Prometheus](https://prometheus.io/), [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) and [Grafana](https://grafana.com/) stack for a complete monitoring setup of both the Kubernetes cluster itself and applications running on it.

- Prometheus does the _service monitoring_ and keeps time-series data. There's one for the cluster monitoring and another one for monitoring your applications.
- Alertmanager is responsible for handling alerts, based on _Rules_ from Prometheus. In our case Alertmanager is responsible for making sure alerts end up in [Opsgenie](https://www.opsgenie.com/) and Slack.
- Grafana provides nice charts and dashboards of the Prometheus time-series data.

We make use of the CoreOS _operator_ principle: by deploying [prometheus-operator](https://github.com/coreos/prometheus-operator) we define new Kubernetes Custom Resource Definitions (CRD) for `Prometheus`, `Alertmanager`, `ServiceMonitor` and `PrometheusRule` which are responsible for deploying Prometheus and Alertmanager setups, Prometheus scrape targets and alerting rules, respectively. As of this writing there isn't an operator setup yet for Grafana.

# Kubernetes application monitoring

As mentioned before, for application monitoring we deploy a separate [Prometheus](https://prometheus.io/) setup, but we reuse the Alertmanager and Grafana from the cluster monitoring.

## Custom config

You can add extra `ServiceMonitor` and `PrometheusRule` objects to monitor and create alerts for your applications. Below we provide some examples. The only requirements are that those objects need to be created in the same namespace as the Prometheus that has to pick them up (`application-monitoring`), and they need to have the following labels:

```yaml
app: prometheus
prometheus: client-monitor
```

*Note: even if those objects are created in the `application-monitoring` namespace, Prometheus can scrape targets in different namespaces.*

### ServiceMonitors

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

You can find more examples in the official Prometheus operator repository: https://github.com/coreos/prometheus-operator/blob/master/example/user-guides/getting-started/example-app-service-monitor.yaml

### Alerting Rules

An example of `PrometheusRule` would be:

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
