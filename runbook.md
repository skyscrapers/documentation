# Runbook

This is the Skyscrapers alerts runbook (or playbook), inspired in the [kubernetes-mixin own runbook](https://github.com/kubernetes-monitoring/kubernetes-mixin/blob/master/runbook.md).

In this page you should find detailed information about specific alerts comming from your monitoring system. It's possible though that some alerts haven't been documented yet or the information is incomplete or outdated. If you find a missing alert or inacurate information, feel free to submit an issue or pull request.

In addition of the alerts listed in this page, there are other system alerts that are described in upstream runbooks, like the one linked above. Always follow the `runbook_url` annotation link (:notebook:) in the alert notification to get the most recent and up-to-date information about that alert.

## Kubernetes alerts

### Alert Name: CalicoNodeInstanceDown

* *Description*: `{{$labels.instance}} of job {{$labels.job}} has been down for more than 5 minutes`
* *Severity*: `critical`
* *Action*: Check the `calico-node` Pods with `kubectl get pods -n kube-system -l k8s-app=calico-node` and look at the Pod logs using `kubectl logs -n kube-system <pod>` for further information.

### Alert Name: CalicoDataplaneFailures

* *Description*: `{{$labels.instance}} with calico-node pod {{$labels.pod}} has been having dataplane failures`
* *Severity*: `warning`
* *Action*: Check the `calico-node` Pods with `kubectl get pods -n kube-system -l k8s-app=calico-node` and look at the Pod logs using `kubectl logs -n kube-system <pod>` for further information.

### Alert Name: MemoryOvercommitted

* *Description*: `{{$labels.node}} is overcommited by {{$value}}%`
* *Severity*: `warning`
* *Action*: Check the actual memory usage of the cluster. If it's not too high it might indicate that Pod resource requests are not adjusted properly. Otherwise try adding more nodes to the cluster.

### Alert Name: CPUUsageHigh

* *Description*: `{{$labels.instance}} is using more than 90% CPU for >1h`
* *Severity*: `warning`

## ElasticSearch alerts

### Alert Name: ElasticsearchExporterDown

* *Description*: `The Elasticsearch metrics exporter for {{ $labels.job }} is down!`
* *Severity*: `critical`
* *Action*: Check the `elasticsearch-exporter` Pods with `kubectl get pods -n infrastructure -l app=elasticsearch-exporter,release=logging-es-monitor` and look at the Pod logs using `kubectl logs -n infrastructure <pod>` for further information.

### Alert Name: ElasticsearchCloudwatchExporterDown

* *Description*: `The Elasticsearch Cloudwatch metrics exporter for {{ $labels.job }} is down!`
* *Severity*: `critical`
* *Action*: Check the `cloudwatch-exporter` Pods with `kubectl get pods -n infrastructure -l app=prometheus-cloudwatch-exporter,release=logging-es-monitor` and look at the Pod logs using `kubectl logs -n infrastructure <pod>` for further information.

### Alert Name: ElasticsearchClusterHealthYellow

* *Description*: `Elasticsearch cluster health for {{ $labels.cluster }} is yellow.`
* *Severity*: `warning`

### Alert Name: ElasticsearchClusterHealthRed

* *Description*: `Elasticsearch cluster health for {{ $labels.cluster }} is RED!`
* *Severity*: `critical`

### Alert Name: ElasticsearchClusterEndpointDown

* *Description*: `Elasticsearch cluster endpoint for {{ $labels.cluster }} is DOWN!`
* *Severity*: `critical`

### Alert Name: ElasticsearchAWSLowDiskSpace

* *Description*: `AWS Elasticsearch cluster {{ $labels.domain_namae }} has less than 10GB left`
* *Severity*: `warning`

### Alert Name: ElasticsearchAWSNoDiskSpace

* *Description*: `AWS Elasticsearch cluster {{ $labels.domain_name }} has less than 5GB left`
* *Severity*: `critical`

### Alert Name: ElasticsearchLowDiskSpace

* *Description*: `Elasticsearch cluster {{ $labels.domain_namae }} has less than 10GB left`
* *Severity*: `warning`

### Alert Name: ElasticsearchNoDiskSpace

* *Description*: `Elasticsearch cluster {{ $labels.domain_name }} has less than 5GB left`
* *Severity*: `critical`

### Alert Name: ElasticsearchHeapTooHigh

* *Description*: `The JVM heap usage for Elasticsearch cluster {{ $labels.cluster }} on node {{ $labels.node }} has been over 90% for 15m`
* *Severity*: `warning`

## MongoDB alerts

### Alert Name: MongodbMetricsDown

* *Description*: `The MongoDB metrics exporter for {{ $labels.job }} is down!`
* *Severity*: `critical`
* *Action*: The MongoDB metrics exporter is running on the MongoDB servers. Check the `mongodb-monitoring-metrics` service to know which endpoints it is targetting by using `kubectl describe service -n infrastructure mongodb-monitoring-metrics`.

### Alert Name: MongodbNoConnectionsAvailable

* *Description*: `No connections available anymore on {{$labels.instance}}`
* *Severity*: `warning`

### Alert Name: MongodbUnhealthyMember

* *Description*: `A mongo node with issues has been detected on {{$labels.instance}}`
* *Severity*: `critical`

### Alert Name: MongodbReplicationLagWarning

* *Description*: `The replication is running out on {{$labels.instance}} more than 10 seconds`
* *Severity*: `warning`

### Alert Name: MongodbReplicationLagCritical

* *Description*: `The replication is running out on {{$labels.instance}} more than 60 seconds`
* *Severity*: `critical`

## Other Kubernetes Runbooks and troubleshooting

* [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)
* [Recover a Broken Cluster](https://codefresh.io/Kubernetes-Tutorial/recover-broken-kubernetes-cluster/)
