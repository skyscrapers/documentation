# Runbook

This is the Skyscrapers alerts runbook (or playbook), inspired in the upstream [kubernetes-mixin own runbook](https://github.com/kubernetes-monitoring/kubernetes-mixin/blob/master/runbook.md).

In this page you should find detailed information about specific alerts comming from your monitoring system. It's possible though that some alerts haven't been documented yet or the information is incomplete or outdated. If you find a missing alert or inacurate information, feel free to submit an issue or pull request.

In addition to the alerts listed on this page, there are other system alerts that are described in upstream runbooks, like the one linked above. Always follow the `runbook_url` annotation link (:notebook:) in the alert notification to get the most recent and up-to-date information about that alert.

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

* *Description*: `AWS Elasticsearch cluster {{ $labels.cluster }} is low on free disk space`
* *Severity*: `warning`
* *Extended description*: the amount of free space is computed like `free_space / max(available_space, 100GB)`. So if an ES cluster has a disk size of 1TB, this alert will go off when the free space goes below 10GB. But if the disk size is 50GB, the alert will go off when the free space drops below 5GB.

### Alert Name: ElasticsearchAWSNoDiskSpace

* *Description*: `AWS Elasticsearch cluster {{ $labels.cluster }} has no free disk space`
* *Severity*: `critical`
* *Extended description*: the amount of free space is computed like `free_space / max(available_space, 100GB)`. So if an ES cluster has a disk size of 1TB, this alert will go off when the free space goes below 5GB. But if the disk size is 50GB, the alert will go off when the free space drops below 2.5GB.

### Alert Name: ElasticsearchLowDiskSpace

* *Description*: `Elasticsearch node {{ $labels.node }} on cluster {{ $labels.cluster }} is low on free disk space`
* *Severity*: `warning`

### Alert Name: ElasticsearchNoDiskSpace

* *Description*: `Elasticsearch node {{ $labels.node }} on cluster {{ $labels.cluster }} has no free disk space`
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

## Redshift alerts

### Alert Name: RedshiftExporterDown

* *Description*: `The Redshift metrics exporter for {{ $labels.job }} is down!`
* *Severity*: `critical`
* *Action*: Check the `Redshift-exporter` Pods with `kubectl get pods -n infrastructure -l app=redshift-exporter,release=logging-redshift-monitor` and look at the Pod logs using `kubectl logs -n infrastructure <pod>` for further information.

### Alert Name: RedshiftHealthStatus

* *Description*: `Redshift cluster is not healthy for {{`{{ $labels.cluster }}`}}!`
* *Severity*: `critical`
* *Action*: logon to the aws account and open the Redshift dashboard. Investigate why the cluster is not healthy and take action

### Alert Name: RedshiftMaintenanceMode

* *Description*: `Redshift cluster is in maintenance mode for {{`{{ $labels.cluster }}`}}!`
* *Severity*: `warning`
* *Action*: Maintenance mode is active for the cluster (this should normally be planned and expected). Cluster availability might be impacted.

### Alert Name: RedshiftLowDiskSpace

* *Description*: `AWS Redshift cluster {{`{{ $labels.cluster }}`}} is low on free disk space`
* *Severity*: `warning`
* *Action*: Disk space is running low on the cluster. Check off if this is expected and take action to increase the disk space together with the lead engineer and the customer.

### Alert Name: RedshiftNoDiskSpace

* *Description*: `AWS Redshift cluster {{`{{ $labels.cluster }}`}} is out of free disk space`
* *Severity*: `critical`
* *Action*: There is no disk space left on the cluster. Take immediate action and increase storage on the cluster.

### Alert Name: RedshiftCPUHigh

* *Description*: `AWS Redshift cluster {{`{{ $labels.cluster }}`}} is running at max CPU for 30 minutes`
* *Severity*: `warning`
* *Action*: The cluster is running at max CPU for at least 30 minutes. Check what causes this together with the customer and if needed take action.

### Alert Name VaultIsSealed
* *Description*: `Vault is sealed and unable to auto-unseal`
* *Severity*: `critical`
* *Action*: The Vault cluster normally unseals automatically. Now however the cluster is locked down and unable to unseal itself. Check the logs to see whats going wrong.
