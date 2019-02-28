# Runbook

This is the Skyscrapers alerts runbook (or playbook), inspired in the [kubernetes-mixin own runbook](https://github.com/kubernetes-monitoring/kubernetes-mixin/blob/master/runbook.md).

In this page you should find detailed information about specific alerts comming from your monitoring system. It's possible though that some alerts haven't been documented yet or the information is incomplete or outdated. If you find a missing alert or inacurate information, feel free to submit an issue or pull request.

In addition of the alerts listed in this page, there are other system alerts that are described in upstream runbooks, like the one linked above. Always follow the `runbook_url` annotation link (:notebook:) in the alert notification to get the most recent and up-to-date information about that alert.

## Kubernetes alerts

### Alert Name: CalicoNodeInstanceDown

### Alert Name: CalicoDataplaneFailures

### Alert Name: MemoryOvercommitted

### Alert Name: CPUUsageHigh

## ElasticSearch alerts

### Alert Name: ElasticsearchExporterDown

### Alert Name: ElasticsearchCloudwatchExporterDown

### Alert Name: ElasticsearchClusterHealthYellow

### Alert Name: ElasticsearchClusterHealthRed

### Alert Name: ElasticsearchClusterEndpointDown

### Alert Name: ElasticsearchAWSLowDiskSpace

### Alert Name: ElasticsearchAWSNoDiskSpace

### Alert Name: ElasticsearchLowDiskSpace

### Alert Name: ElasticsearchNoDiskSpace

### Alert Name: ElasticsearchHeapTooHigh

## MongoDB alerts

### Alert Name: MongodbMetricsDown

### Alert Name: MongodbNoConnectionsAvailable

### Alert Name: MongodbUnhealthyMember

### Alert Name: MongodbReplicationLagWarning

### Alert Name: MongodbReplicationLagCritical

## Other Kubernetes Runbooks and troubleshooting

* [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)
* [Recover a Broken Cluster](https://codefresh.io/Kubernetes-Tutorial/recover-broken-kubernetes-cluster/)
