# Kubernetes monitoring

We use a [Prometheus](https://prometheus.io/), [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) and [Grafana](https://grafana.com/) stack for a complete monitoring setup of both the Kubernetes cluster itself and applications running on it.

- Prometheus does the _service monitoring_ and keeps time-series data. There's one for the cluster monitoring and another one for monitoring your applications.
- Alertmanager is responsible for handling alerts, based on _rules_ from Prometheus. In our case Alertmanager is responsible for making sure alerts end up in [Opsgenie](https://www.opsgenie.com/) and Slack.
- Grafana provides nice charts and dashboards of the Prometheus time-series data.

We make use of the CoreOS _operator_ principle: by deploying [prometheus-operator](https://github.com/coreos/prometheus-operator) we define new Kubernetes Custom Resource Definitions (CRD) for `Prometheus`, `Alertmanager`, `ServiceMonitor` and `PrometheusRule` which are responsible for deploying Prometheus and Alertmanager setups, Prometheus scrape targets and alerting rules, respectively.

As of this writing there isn't an operator setup yet for Grafana, but you can add custom dashboards dynamically via `ConfigMaps`.

## Kubernetes application monitoring

By default we create an `application-monitoring` namespace where you can deploy your application's `ServiceMonitors` and `PrometheusRules` and these new configs will be picked up automatically by Prometheus. The only extra requirement is that these resources also need a `prometheus` label set (any value will do).

If you wish, it is also possible to include any other namespace to be searched for Prometheus configs by adding a `prometheus` label to the namespace (any value will do). For example: `kubectl label namespace default prometheus=true`.

Custom Grafana Dashboards can also be added to this same namespace by creting a `ConfigMap` with a `grafana_dashboard` label (any value will do), containing the dashboard's json data. Please make sure that `"datasources"` are set to `"Prometheus"` in your dashboards!

*Note: even if these objects are created in the `application-monitoring` namespace, Prometheus can scrape metrics targets in all namespaces.*

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

You can find more examples in the official [Prometheus Operator documentation](https://github.com/coreos/prometheus-operator/tree/master/Documentation/user-guides)

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
```

You can find more examples in the official [Prometheus Operator documentation](https://github.com/coreos/prometheus-operator/tree/master/Documentation/user-guides)

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

### Prometheus scrapers for common technologies

Here's a list of Prometheus scrapers already available for common frameworks.

#### PHP

##### Native

Prometheus has a [native client library](https://github.com/Jimdo/prometheus_client_php) for PHP. This is easy to implement inside your application, but by default doesn't expose standard metrics. This is ideal if you want to expose your own metrics (can also be business metrics).

##### PHP-FPM

If you want to get PHP-FPM metrics, we recommend using [this exporter](https://github.com/hipages/php-fpm_exporter). It's being actively maintained and the documentation is reasonably good. You have to setup the exporter as a sidecar container in your pods, then it'll access the PHP-FPM socket to read statistics and expose them as Prometheus metrics.

You first need to expose the metrics in PHP-FPM. You can do this by adding the following config your PHP-FPM image.

```
pm.status_path = /status
```

Then you'll need to add the `php-fpm-exporter` as a sidecar container to your pod, like this:

```
        - name: {{ template "app.fullname" . }}-fpm-exporter
          image: "hipages/php-fpm_exporter:0.5.2"
          command: ["--phpfpm.scrape-uri", "tcp://127.0.0.1:{{ .Values.app.port }}/status"]
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
```

*Note that you'll need to adjust `{{ template "app.fullname" . }}` and `{{ .Values.app.port }}` to the correct helm variables. The first one represents the app name we want to monitor. The second is the php-fpm port of the application.*

After that, you can add a `ServiceMonitor` to scrape it.

#### Ruby

##### Native

Prometheus has a [native client library](https://github.com/prometheus/client_ruby) for Ruby. This is a really good library when you run your ruby application as single process. Unfortunately a lot of applications use Unicorn or another multi-process integration. When using a multi-process Ruby you can best use the [fork](https://gitlab.com/gitlab-org/prometheus-client-mmap) of GitLab.

You have to integrate this library in your application and expose it as an endpoint. Once that is done, you can add a `ServiceMonitor` to scrape it.

##### Workers

Workers by default don't expose a webserver to scrape from. This will have to change and every worker will need to expose a simple webserver so that Prometheus can scrape its metrics.

It is really discouraged to use Pushgateway for this. For more info why this is discouraged, see the [Pushgateway documentation](https://prometheus.io/docs/practices/pushing/#should-i-be-using-the-pushgateway).

#### Nginx

Nginx has an exporter through the [Nginx VTS module](https://github.com/vozlt/nginx-module-vts). We [forked](https://github.com/skyscrapers/docker-images/tree/master/nginx) the official Docker Nginx image and added the VTS module to nginx. We had to fork it because the VTS module needs to be compiled with Nginx.

##### Setup

1. Use our [Nginx](https://hub.docker.com/r/skyscrapers/nginx/) Docker image instead of the upstream Nginx.
2. Add the [exporter](https://github.com/hnlq715/nginx-vts-exporter) as sidecar to the pod.

```
        - name: {{ template "app.fullname" . }}-exporter
          image: "sophos/nginx-vts-exporter:v0.10.0"
          env:
            - name: NGINX_STATUS
              value: http://localhost:9999/status/format/json
          ports:
            - name: prom
              containerPort: 9913
```

*Note that you will need to adjust `{{ template "app.fullname" . }}` to the correct helm variables. It represents the app name you want to monitor.*

After that, you can add a `ServiceMonitor` to scrape it.

## Default alerts

This is a list of all the alerts that are configured by default in your Kubernetes cluster.

| Name                                   | Description                                                                                                                                                                                                         |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DeadMansSwitch                         | This is a DeadMansSwitch meant to ensure that the entire Alerting pipeline is functional. If active it means it's working fine                                                                                      |
| DeploymentReplicasNotUpdated           | A deployment has not been rolled out properly. Either replicas are not being updated to the most recent version, or not all replicas are ready. The alert does not fire if the deployment was paused intentionally. |
| LinkerdHighLatency                     | the time-to-first-byte (the elapsed time between the proxy processing a request’s headers and the first data frame of the response request) is higher than 2secs                                                   |
| APIServerErrorsHigh                    | The API server responds to a lot of requests with errors. > 2 returns a warning, >5 returns a critical                                                                                                              |
| APIServerLatencyHigh                   | The response latency of the API server to clients is high. warning > 1s, critical > 4s                                                                                                                              |
| AlertmanagerConfigInconsistent         | The configuration of the instances of the Alertmanager cluster are out of sync.                                                                                                                                     |
| AlertmanagerDownOrMissing              | Alertmanager is down or not present                                                                                                                                                                                 |
| AlertmanagerFailedReload               | Alertmanager not able to reload                                                                                                                                                                                     |
| DaemonSetRolloutStuck                  | A daemon set is not fully rolled out to all desired nodes.                                                                                                                                                          |
| DaemonSetsMissScheduled                | A number of daemonsets are running where they are not supposed to run.                                                                                                                                              |
| DeploymentGenerationMismatch           | The observed generation of a deployment does not match its desired generation. (“Generation” is a number that increments with each deployment)                                                                    |
| ElasticsearchCloudwatchExporterDown    | The Elasticsearch Cloudwatch metrics exporter is down!                                                                                                                                                              |
| ElasticsearchClusterEndpointDown       | Elasticsearch cluster endpoint is DOWN!                                                                                                                                                                             |
| ElasticsearchClusterHealthRed          | Elasticsearch cluster health is RED!                                                                                                                                                                                |
| ElasticsearchClusterHealthYellow       | Elasticsearch cluster health is yellow.                                                                                                                                                                             |
| ElasticsearchExporterDown              | The Elasticsearch metrics exporter for the specified job is down!                                                                                                                                                   |
| ElasticsearchHeapTooHigh               | The JVM heap usage for Elasticsearch cluster has been over 90% for 15m                                                                                                                                              |
| ElasticsearchLowDiskSpace              | disk space in elasticsearch node is <10GB                                                                                                                                                                           |
| ElasticsearchNoDiskSpace               | disk space in elasticsearch node is <5GB                                                                                                                                                                            |
| EtcdMemberCommunicationSlow            | an etcd instance member communication is slow                                                                                                                                                                       |
| FdExhaustionClose                      | instance will exhaust in file descriptors within the next 4hour (warning) or 1hour (critical)                                                                                                                       |
| GRPCRequestsSlow                       | gRPC requests are slow for an etcd instance                                                                                                                                                                         |
| HTTPRequestsSlow                       | HTTP requests are slow for an etcd instance                                                                                                                                                                         |
| HighCommitDurations                    | commit durations are high for an etcd instance                                                                                                                                                                      |
| HighFsyncDurations                     | file sync durations are high for an etcd instance                                                                                                                                                                   |
| HighNumberOfFailedGRPCRequests         | etcd grpc requests have a failure rate > 0.01% for warning and 0.05% for critical                                                                                                                                   |
| HighNumberOfFailedHTTPRequests         | etcd requests have a failure rate > 0.01% for warning and 0.05% for critical                                                                                                                                        |
| HighNumberOfFailedProposals            | an etcd instance has seen >5 proposal failures within the last hour                                                                                                                                                 |
| HighNumberOfLeaderChanges              | an etcd instance has seen >3 leader changes within the last hour                                                                                                                                                    |
| InsufficientMembers                    | If one more etcd member goes down the cluster will be unavailable                                                                                                                                                   |
| K8SApiserverDown                       | The API server is unreachable. Prometheus failed to scrape the API server(s), or all API servers have disappeared from service discovery.                                                                           |
| K8SControllerManagerDown               | There is no running K8S controller manager. Deployments and replication controllers are not making progress.                                                                                                        |
| K8SDaemonSetsNotScheduled              | A number of daemonsets are not scheduled                                                                                                                                                                            |
| K8SKubeletDown                         | Many kubelets cannot be scraped. Prometheus failed to scrape the listed percentage of kubelets, or all kubelets have disappeared from service discovery.                                                            |
| K8SKubeletTooManyPods                  | Kubelet is close to pod limit. The given kubelet instance is running the listed number of pods, which is close to the limit of 110.                                                                                 |
| K8SManyNodesNotReady                   | More than 10% of the listed number of Kubernetes nodes are NotReady.                                                                                                                                                |
| K8SNodeNotReady                        | The Kubelet on the listed node has not checked in with the API, or has set itself to NotReady, for more than an hour.                                                                                               |
| K8SSchedulerDown                       | There is no running Kubernetes scheduler. New pods are not being assigned to nodes.                                                                                                                                 |
| K8sCertificateExpirationNotice         | Kubernetes API Certificate is expiring soon (less than 1 day warning, less than 7 days critical)                                                                                                                    |
| NoLeader                               | No leader selected for etcd                                                                                                                                                                                         |
| NodeDiskRunningFull                    | If disks keep filling up at the current pace they will run out of free space within the next hours.                                                                                                                 |
| NodeExporterDown                       | Prometheus could not scrape a node-exporter for more than 10m, or node-exporters have disappeared from discovery.                                                                                                   |
| PodFrequentlyRestarting                | A pod is restarting several times an hour.                                                                                                                                                                          |
| PrometheusConfigReloadFailed           | Reloading Prometheus' configuration has failed                                                                                                                                                                      |
| PrometheusErrorSendingAlerts           | A monitored Prometheus instance is not able to send alerts to Alertmanager (> 0.01% Warning, > 0.03% Critical )                                                                                                     |
| PrometheusNotConnectedToAlertmanagers  | A monitored Prometheus instance is not connected to any Alertmanagers. Any firing alerts will not be sent anywhere.                                                                                                 |
| PrometheusNotIngestingSamples          | Prometheus isn't able to ingest samples.                                                                                                                                                                            |
| PrometheusNotificationQueueRunningFull | Prometheus is generating more alerts than it can send to Alertmanagers in time.                                                                                                                                     |
| PrometheusTSDBCompactionsFailing       | Total number of WAL (write-ahead-log) compaction failures on the TSDB database over the first 4 hours                                                                                                               |
| PrometheusTSDBReloadsFailing           | Total number of WAL (write-ahead-log) reload failures on the TSDB database over the first 4 hours                                                                                                                   |
| PrometheusTSDBWALCorruptions           | Total number of WAL (write-ahead-log) corruptions on the TSDB database > 0                                                                                                                                          |
| TargetDown                             | Prometheus scrape targets are down. The listed percentage of job targets are down.                                                                                                                                  |
