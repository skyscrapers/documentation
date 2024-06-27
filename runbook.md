# Runbook

This is the Skyscrapers alerts runbook (or playbook), inspired by the upstream [kubernetes-mixin runbook](https://github.com/kubernetes-monitoring/kubernetes-mixin/blob/master/runbook.md).

On this page you should find detailed information about specific alerts comming from your monitoring system. It's possible though that some alerts haven't been documented yet or the information is incomplete or outdated. If you find a missing alert or inacurate information, feel free to submit an issue or pull request.

In addition to the alerts listed on this page, there are other system alerts that are described in upstream runbooks, like the one linked above. Always follow the `runbook_url` annotation link (:notebook:) in the alert notification to get the most recent and up-to-date information about that alert.

- [Runbook](#runbook)
  - [Kubernetes alerts](#kubernetes-alerts)
    - [Alert Name: MemoryOvercommitted](#alert-name-memoryovercommitted)
    - [Alert Name: CPUUsageHigh](#alert-name-cpuusagehigh)
    - [Alert Name: NodeWithImpairedVolumes](#alert-name-nodewithimpairedvolumes)
    - [Alert Name: ContainerExcessiveCPU](#alert-name-containerexcessivecpu)
    - [Alert Name: ContainerExcessiveMEM](#alert-name-containerexcessivemem)
  - [ElasticSearch alerts](#elasticsearch-alerts)
    - [Alert Name: ElasticsearchExporterDown](#alert-name-elasticsearchexporterdown)
    - [Alert Name: ElasticsearchCloudwatchExporterDown](#alert-name-elasticsearchcloudwatchexporterdown)
    - [Alert Name: ElasticsearchClusterHealthYellow](#alert-name-elasticsearchclusterhealthyellow)
    - [Alert Name: ElasticsearchClusterHealthRed](#alert-name-elasticsearchclusterhealthred)
    - [Alert Name: ElasticsearchClusterEndpointDown](#alert-name-elasticsearchclusterendpointdown)
    - [Alert Name: ElasticsearchAWSLowDiskSpace](#alert-name-elasticsearchawslowdiskspace)
    - [Alert Name: ElasticsearchAWSNoDiskSpace](#alert-name-elasticsearchawsnodiskspace)
    - [Alert Name: ElasticsearchIndexWritesBlocked](#alert-name-elasticsearchindexwritesblocked)
    - [Alert Name: ElasticsearchLowDiskSpace](#alert-name-elasticsearchlowdiskspace)
    - [Alert Name: ElasticsearchNoDiskSpace](#alert-name-elasticsearchnodiskspace)
    - [Alert Name: ElasticsearchHeapTooHigh](#alert-name-elasticsearchheaptoohigh)
  - [Fluent Bit alerts](#fluent-bit-alerts)
    - [Alert Name: FluentbitDroppedRecords](#alert-name-fluentbitdroppedrecords)
  - [Loki alerts](#loki-alerts)
    - [Alert Name: LokiDiscardingSamples](#alert-name-lokidiscardingsamples)
    - [Alert Name: LokiNotFlushingChunks](#alert-name-lokinotflushingchunks)
  - [MongoDB alerts](#mongodb-alerts)
    - [Alert Name: MongodbMetricsDown](#alert-name-mongodbmetricsdown)
    - [Alert Name: MongodbLowConnectionsAvailable](#alert-name-mongodblowconnectionsavailable)
    - [Alert Name: MongodbUnhealthyMember](#alert-name-mongodbunhealthymember)
    - [Alert Name: MongodbReplicationLagWarning](#alert-name-mongodbreplicationlagwarning)
    - [Alert Name: MongodbReplicationLagCritical](#alert-name-mongodbreplicationlagcritical)
  - [RDS alerts](#rds-alerts)
    - [Alert Name: RDSCPUCreditBalanceLow](#alert-name-rdscpucreditbalancelow)
    - [Alert Name: RDSFreeableMemoryLow](#alert-name-rdsfreeablememorylow)
    - [Alert Name: RDSFreeStorageSpaceRunningLow](#alert-name-rdsfreestoragespacerunninglow)
    - [Alert Name: RDSDiskQueueDepthHigh](#alert-name-rdsdiskqueuedepthhigh)
    - [Alert Name: RDSCPUUsageHigh](#alert-name-rdscpuusagehigh)
    - [Alert Name: RDSReplicaLagHigh](#alert-name-rdsreplicalaghigh)
    - [Alert Name: RDSBurstBalanceLow](#alert-name-rdsburstbalancelow)
  - [Concourse alerts](#concourse-alerts)
    - [ConcourseWorkersMismatch](#concourseworkersmismatch)
    - [ConcourseWorkerCPUCreditBalanceLow](#concourseworkercpucreditbalancelow)
    - [ConcourseWorkerEBSIOBalanceLow](#concourseworkerebsiobalancelow)
    - [Aler Name: ConcourseEndpointDown](#aler-name-concourseendpointdown)
  - [Redshift alerts](#redshift-alerts)
    - [Alert Name: RedshiftExporterDown](#alert-name-redshiftexporterdown)
    - [Alert Name: RedshiftHealthStatus](#alert-name-redshifthealthstatus)
    - [Alert Name: RedshiftMaintenanceMode](#alert-name-redshiftmaintenancemode)
    - [Alert Name: RedshiftLowDiskSpace](#alert-name-redshiftlowdiskspace)
    - [Alert Name: RedshiftNoDiskSpace](#alert-name-redshiftnodiskspace)
    - [Alert Name: RedshiftCPUHigh](#alert-name-redshiftcpuhigh)
    - [Alert Name VaultIsSealed](#alert-name-vaultissealed)
  - [cert-manager alerts](#cert-manager-alerts)
    - [Alert Name: CertificateNotReady](#alert-name-certificatenotready)
    - [Alert Name: CertificateAboutToExpire](#alert-name-certificateabouttoexpire)
    - [Alert Name: AmazonMQCWExporterDown](#alert-name-amazonmqcwexporterdown)
    - [Alert Name: AmazonMQMemoryAboveLimit](#alert-name-amazonmqmemoryabovelimit)
    - [Alert Name: AmazonMQDiskFreeBelowLimit](#alert-name-amazonmqdiskfreebelowlimit)
  - [ExternalDNS alerts](#externaldns-alerts)
    - [Alert Name: ExternalDnsRegistryErrorsIncrease](#alert-name-externaldnsregistryerrorsincrease)
    - [Alert Name: ExternalDNSSourceErrorsIncrease](#alert-name-externaldnssourceerrorsincrease)
  - [Velero alerts](#velero-alerts)
    - [Alert Name: VeleroBackupPartialFailures](#alert-name-velerobackuppartialfailures)
    - [Alert Name: VeleroBackupFailures](#alert-name-velerobackupfailures)
    - [Alert Name: VeleroVolumeSnapshotFailures](#alert-name-velerovolumesnapshotfailures)
    - [Alert Name: VeleroBackupTooOld](#alert-name-velerobackuptooold)
  - [VPA alerts](#vpa-alerts)
    - [Alert Name: VPAAdmissionControllerDown](#alert-name-vpaadmissioncontrollerdown)
    - [Alert Name: VPAAdmissionControllerSlow](#alert-name-vpaadmissioncontrollerslow)
    - [Alert Name: VPARecommenderDown](#alert-name-vparecommenderdown)
    - [Alert Name: VPARecommenderSlow](#alert-name-vparecommenderslow)
    - [Alert Name: VPAUpdaterDown](#alert-name-vpaupdaterdown)
    - [Alert Name: VPAUpdaterSlow](#alert-name-vpaupdaterslow)
  - [Other Kubernetes Runbooks and troubleshooting](#other-kubernetes-runbooks-and-troubleshooting)

## Kubernetes alerts

### Alert Name: MemoryOvercommitted

- *Description*: `{{$labels.node}} is overcommited by {{$value}}%`
- *Severity*: `warning`
- *Action*: Check the actual memory usage of the cluster. If it's not too high it might indicate that Pod resource requests are not adjusted properly. Otherwise try adding more nodes to the cluster.

### Alert Name: CPUUsageHigh

- *Description*: `{{$labels.instance}} is using more than 90% CPU for >1h`
- *Severity*: `warning`

### Alert Name: NodeWithImpairedVolumes

- *Description*: `EBS volumes are failing to attach to node {{$labels.node}}`
- *Severity*: `critical`
- *Action*: Check the AWS Console which volumes are stuck in an `Attaching` state. Unattach the volume and delete the Pod the volume belongs to (most likely in `Pending` state) so it can be rescheduled on another node. Untaint the affected node when everything is OK again.

### Alert Name: ContainerExcessiveCPU

- *Description*: `Container {{ $labels.container }} of pod {{ $labels.namespace }}/{{ $labels.pod }} has been using {{ $value | humanizePercentage }} of it's requested CPU for 30 min`
- *Severity*: `warning`
- *Action*: Check resource usage of the container through `kubectl top` or Grafana. Consider increasing the container's CPU request and/or setting a limit.

### Alert Name: ContainerExcessiveMEM

- *Description*: `Container {{ $labels.container }} of pod {{ $labels.namespace }}/{{ $labels.pod }} has been using {{ $value | humanizePercentage }} of it's requested Memory for 30 min`
- *Severity*: `warning`
- *Action*: Check resource usage of the container through `kubectl top` or Grafana. Consider increasing the container's Memory request and/or setting a limit.

## ElasticSearch alerts

### Alert Name: ElasticsearchExporterDown

- *Description*: `The Elasticsearch metrics exporter for {{ $labels.job }} is down!`
- *Severity*: `critical`
- *Action*: Check the `elasticsearch-exporter` Pods with `kubectl get pods -n infrastructure -l app=elasticsearch-exporter,release=logging-es-monitor` and look at the Pod logs using `kubectl logs -n infrastructure <pod>` for further information.

### Alert Name: ElasticsearchCloudwatchExporterDown

- *Description*: `The Elasticsearch Cloudwatch metrics exporter for {{ $labels.job }} is down!`
- *Severity*: `critical`
- *Action*: Check the `cloudwatch-exporter` Pods with `kubectl get pods -n infrastructure -l app=prometheus-cloudwatch-exporter,release=logging-es-monitor` and look at the Pod logs using `kubectl logs -n infrastructure <pod>` for further information.

### Alert Name: ElasticsearchClusterHealthYellow

- *Description*: `Elasticsearch cluster health for {{ $labels.cluster }} is yellow.`
- *Severity*: `warning`

### Alert Name: ElasticsearchClusterHealthRed

- *Description*: `Elasticsearch cluster health for {{ $labels.cluster }} is RED!`
- *Severity*: `critical`

### Alert Name: ElasticsearchClusterEndpointDown

- *Description*: `Elasticsearch cluster endpoint for {{ $labels.cluster }} is DOWN!`
- *Severity*: `critical`

### Alert Name: ElasticsearchAWSLowDiskSpace

- *Description*: `AWS Elasticsearch cluster {{ $labels.cluster }} is low on free disk space`
- *Severity*: `warning`
- *Extended description*: the amount of free space is computed like `free_space / max(available_space, 100GB)`. So if an ES cluster has a disk size of 1TB, this alert will go off when the free space goes below 10GB. But if the disk size is 50GB, the alert will go off when the free space drops below 5GB.

### Alert Name: ElasticsearchAWSNoDiskSpace

- *Description*: `AWS Elasticsearch cluster {{ $labels.cluster }} has no free disk space`
- *Severity*: `critical`
- *Extended description*: the amount of free space is computed like `free_space / max(available_space, 100GB)`. So if an ES cluster has a disk size of 1TB, this alert will go off when the free space goes below 5GB. But if the disk size is 50GB, the alert will go off when the free space drops below 2.5GB.

### Alert Name: ElasticsearchIndexWritesBlocked

- *Description*: `AWS Elasticsearch cluster {{ $labels.cluster }} is blocking incoming write requests`
- *Severity*: `critical`
- *Extended description (from the [AWS official documentation](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains-cloudwatchmetrics.html#es-managedomains-cloudwatchmetrics-cluster-metrics))*: Indicates that the ES cluster is blocking incoming write requests. This alert is only available for AWS ElasticSearch clusters, as it's provided by the Cloudwatch exporter. Some common factors include the following: `FreeStorageSpace` is too low or `JVMMemoryPressure` is too high. To alleviate this issue, consider adding more disk space or scaling your cluster.

### Alert Name: ElasticsearchLowDiskSpace

- *Description*: `Elasticsearch node {{ $labels.node }} on cluster {{ $labels.cluster }} is low on free disk space`
- *Severity*: `warning`

### Alert Name: ElasticsearchNoDiskSpace

- *Description*: `Elasticsearch node {{ $labels.node }} on cluster {{ $labels.cluster }} has no free disk space`
- *Severity*: `critical`

### Alert Name: ElasticsearchHeapTooHigh

- *Description*: `The JVM heap usage for Elasticsearch cluster {{ $labels.cluster }} on node {{ $labels.node }} has been over 90% for 15m`
- *Severity*: `warning`

## Fluent Bit alerts

### Alert Name: FluentbitDroppedRecords

- *Description*: `Fluent Bit {{ $labels.pod }} is failing to save records to output {{ $labels.name }}`
- *Severity*: `critical`
- *Action*: Check the fluent-bit pod's logs for error messages and to see which records are failing to upload.
  - Elasticsearch: In case of Elasticsearch mapping errors, it's possible that the ES HTTP response is cut off in the fluentbit logs. In that case, set [`Buffer_Size False` in the fluent-bit ES `OUTPUT` config](https://docs.fluentbit.io/manual/pipeline/outputs/elasticsearch).
  - Elasticsearch: Make sure your application logs are structured (json recommended) and fields' data types across your applications are consistent.

## Loki alerts

### Alert Name: LokiDiscardingSamples

- *Description*: `Loki is discarding ingested samples for reason {{ $labels.reason }}`
- *Severity*: `critical`
- *Action*: This metric records when an entry was rejected, with a `reason` tag saying why. The reason why the request is dropped is usually due to `rate-limiting` and it is up to the client (Fluent Bit) to retry. This alert is considered critical, because it means logs are lost if the client gives up. In this case you can increase the cluster definition values `spec.grafana_loki.ingestion_rate_mb` and/or `spec.grafana_loki.ingestion_burst_size_mb`, eg. try doubling their size.

### Alert Name: LokiNotFlushingChunks

- *Description*: `Loki writer {{ $labels.pod }} has not flushed any chunks for more than 40 minutes`
- *Severity*: `critical`
- *Action*: This alert could fire due to a problem in the Loki writers themselves, a configuration error with the S3 bucket, or due to an outage in S3. First check the Loki writer Pods for any signs of trouble. Check the Pods logs, if there's a problem connecting to S3 it will show up in the writer logs. Revise the configuration of the components that allow the Pod to access S3 (`ServiceAccount`, IAM role and IAM policy).

## MongoDB alerts

### Alert Name: MongodbMetricsDown

- *Description*: `The MongoDB metrics exporter for {{ $labels.job }} is down!`
- *Severity*: `critical`
- *Action*: The MongoDB metrics exporter is running on the MongoDB servers. Check the `mongodb-monitoring-metrics` service to know which endpoints it is targetting by using `kubectl describe service -n infrastructure mongodb-monitoring-metrics`.

### Alert Name: MongodbLowConnectionsAvailable

- *Description*: `Low connections available on {{$labels.instance}}`
- *Severity*: `warning`
- *Action*: The cluster is running out of connections to accept. This needs to be looked at. Try to find the cause of the buildup of queries (slow queries or something that is going crazy). If the node runs out of connections the replication can come into problems and cause the whole cluster to fail.

### Alert Name: MongodbUnhealthyMember

- *Description*: `A mongo node with issues has been detected on {{$labels.instance}}`
- *Severity*: `critical`

### Alert Name: MongodbReplicationLagWarning

- *Description*: `The replication is running out on {{$labels.instance}} more than 10 seconds`
- *Severity*: `warning`

### Alert Name: MongodbReplicationLagCritical

- *Description*: `The replication is running out on {{$labels.instance}} more than 60 seconds`
- *Severity*: `critical`

## RDS alerts

### Alert Name: RDSCPUCreditBalanceLow

- *Description*: `The CPU credit balance for {{ $labels.dbinstance_identifier }} is less than 5!`
- *Severity*: `critical`

### Alert Name: RDSFreeableMemoryLow

- *Description*: `RDS instance {{ $labels.dbinstance_identifier }} is running low on memory (< 100Mb)!`
- *Severity*: `critical`

### Alert Name: RDSFreeStorageSpaceRunningLow

- *Description*: `Based on recent sampling, disk storage is expected to fill up in four days for RDS instance {{ $labels.dbinstance_identifier }}.`
- *Severity*: `critical`

### Alert Name: RDSDiskQueueDepthHigh

- *Description*: `The number of outstanding IO requests waiting to access the disk is high for RDS instance {{ $labels.dbinstance_identifier }}.`
- *Severity*: `critical`

### Alert Name: RDSCPUUsageHigh

- *Description*: `CPU usage for RDS instance {{ $labels.dbinstance_identifier }} is higher than 95%.`
- *Severity*: `info`

### Alert Name: RDSReplicaLagHigh

- *Description*: `Replica lag for RDS instance {{ $labels.dbinstance_identifier }} is higher than 30 seconds.`
- *Severity*: `warning`

### Alert Name: RDSBurstBalanceLow

- *Description*: `EBS BurstBalance for RDS instance {{ $labels.dbinstance_identifier }} is lower than 20%.`
- *Severity*: `warning`
- *Action*: The amount of IOPS on the disk is too high and the RDS is running out of credits. Check the `RDS` for heavy IO intensive queries (check with the customer). If needed the IOPS need to be increased (can be done by increasing the EBS volume or changing to a PIOPS volume).

## Concourse alerts

### ConcourseWorkersMismatch

- *Description*: `There are stale Concourse workers for more than an hour`
- *Severity*: `critical

### ConcourseWorkerCPUCreditBalanceLow

- *Description*: `Minimum CPU credit balance of one of Concourse workers has reached 0 for an hour`
- *Severity*: `critical

### ConcourseWorkerEBSIOBalanceLow

- *Description*: `EBS IO balance balance of one of Concourse workers volumues has reached 0 for an hour`
- *Severity*: `critical

### Aler Name: ConcourseEndpointDown

- *Description*: `Concourse endpoint has been down for 5 minutes.`
- *Severity*: `critical`

## Redshift alerts

### Alert Name: RedshiftExporterDown

- *Description*: `The Redshift metrics exporter for {{ $labels.job }} is down!`
- *Severity*: `critical`
- *Action*: Check the `Redshift-exporter` Pods with `kubectl get pods -n infrastructure -l app=redshift-exporter,release=logging-redshift-monitor` and look at the Pod logs using `kubectl logs -n infrastructure <pod>` for further information.

### Alert Name: RedshiftHealthStatus

- *Description*: `Redshift cluster is not healthy for {{ $labels.cluster }}!`
- *Severity*: `critical`
- *Action*: logon to the aws account and open the Redshift dashboard. Investigate why the cluster is not healthy and take action

### Alert Name: RedshiftMaintenanceMode

- *Description*: `Redshift cluster is in maintenance mode for {{ $labels.cluster }}!`
- *Severity*: `warning`
- *Action*: Maintenance mode is active for the cluster (this should normally be planned and expected). Cluster availability might be impacted.

### Alert Name: RedshiftLowDiskSpace

- *Description*: `AWS Redshift cluster {{ $labels.cluster }} is low on free disk space`
- *Severity*: `warning`
- *Action*: Disk space is running low on the cluster. Check off if this is expected and take action to increase the disk space together with the lead engineer and the customer.

### Alert Name: RedshiftNoDiskSpace

- *Description*: `AWS Redshift cluster {{ $labels.cluster }} is out of free disk space`
- *Severity*: `critical`
- *Action*: There is no disk space left on the cluster. Take immediate action and increase storage on the cluster.

### Alert Name: RedshiftCPUHigh

- *Description*: `AWS Redshift cluster {{ $labels.cluster }} is running at max CPU for 30 minutes`
- *Severity*: `warning`
- *Action*: The cluster is running at max CPU for at least 30 minutes. Check what causes this together with the customer and if needed take action.

### Alert Name VaultIsSealed

- *Description*: `Vault is sealed and unable to auto-unseal`
- *Severity*: `critical`
- *Action*: The Vault cluster normally unseals automatically. Now however the cluster is locked down and unable to unseal itself. Check the logs to see whats going wrong.

## cert-manager alerts

### Alert Name: CertificateNotReady

- *Description*: `A cert-manager certificate can not be issued/updated`
- *Severity*: `warning`
- *Action*: A certificate fails to be ready within 10 mintues during an issuing or an update event, check the certificate events and the certmanager pod logs to get the reason of the failure.

### Alert Name: CertificateAboutToExpire

- *Description*: `A cert-manager certificate is about to expire`
- *Severity*: `warning`
- *Action*: A certificate has less than x weeks to expire and did not get renewed, check the certificate events and the certmanager pod logs to get the reason of the failure.

### Alert Name: AmazonMQCWExporterDown

- *Description*: `An AmazonMQ for RabbitMQ metrics exporter is down`
- *Severity*: `warning | critical`
- *Action*: Check why the Cloudwatch exporter is failing.

### Alert Name: AmazonMQMemoryAboveLimit

- *Description*: `AmazonMQ for RabbitMQ node memory usage is above the limit. Cluster is now blocking producer connections`
- *Severity*: `warning | critical`
- *Action*: Switch to a bigger instance type for your AmazonMQ broker

### Alert Name: AmazonMQDiskFreeBelowLimit

- *Description*: `AmazonMQ for RabbitMQ node free disk space is lower than the limit. Cluster is now blocking producer connections`
- *Severity*: `warning | critical`
- *Action*: Switch to a bigger instance type for your AmazonMQ broker

## ExternalDNS alerts

### Alert Name: ExternalDnsRegistryErrorsIncrease

- *Description*: `External DNS registry Errors increasing constantly`
- *Severity*: `warning`
- *Action*: `Registry` errors are mostly Provider errors, unless there's some coding flaw in the registry package. Provider errors often arise due to accessing their APIs due to network or missing cloud-provider permissions when reading records. When applying a changeset, errors will arise if the changeset applied is incompatible with the current state. In case of an increased error count, you could correlate them with the `http_request_duration_seconds{handler="instrumented_http"}` metric which should show increased numbers for status codes 4xx (permissions, configuration, invalid changeset) or 5xx (apiserver down). You can use the host label in the metric to figure out if the request was against the Kubernetes API server (Source errors) or the DNS provider API (Registry/Provider errors).

### Alert Name: ExternalDNSSourceErrorsIncrease

- *Description*: `External DNS source Errors increasing constantly`
- *Severity*: `warning`
- *Action*: `Source`s are mostly Kubernetes API objects. Examples of `source` errors may be connection errors to the Kubernetes API server itself or missing RBAC permissions. It can also stem from incompatible configuration in the objects itself like invalid characters, processing a broken fqdnTemplate, etc. In case of an increased error count, you could correlate them with the `http_request_duration_seconds{handler="instrumented_http"}` metric which should show increased numbers for status codes 4xx (permissions, configuration, invalid changeset) or 5xx (apiserver down). You can use the host label in the metric to figure out if the request was against the Kubernetes API server (Source errors) or the DNS provider API (Registry/Provider errors).

## Velero alerts

### Alert Name: VeleroBackupPartialFailures

- *Description*: `Velero backup {{ $labels.schedule }} has {{ $value | humanizePercentage }} partial failures.`
- *Severity*: `warning`
- *Action*: Check whether the `velero` Pod crashes during backup (eg. OOMKill). Check whether velero is trying to backup stale PV/PVCs with a deleted cloud-provider volume (`velero back describe ...` & `velero backup logs ...`).

### Alert Name: VeleroBackupFailures

- *Description*: `Velero backup {{ $labels.schedule }} has {{ $value | humanizePercentage }} failures.`
- *Severity*: `warning`
- *Action*: Check the backup (`velero back describe ...`) and it's logs (`velero backup logs ...`) for the failure reason.

### Alert Name: VeleroVolumeSnapshotFailures

- *Description*: `Velero backup {{ $labels.schedule }} has {{ $value | humanizePercentage }} volume snapshot failures.`
- *Severity*: `warning`
- *Action*: Check the backup (`velero back describe ...`) and it's logs (`velero backup logs ...`) for the failure reason.

### Alert Name: VeleroBackupTooOld

- *Description*: `Cluster hasn't been backed up for more than 3 days.`
- *Severity*: `critical`
- *Action*: This will fire if backups have failed due to any of the above reasons, check the other alerts to figure out what's wrong.

## VPA alerts

### Alert Name: VPAAdmissionControllerDown

- *Description*: `The VPA AdmissionController is down`
- *Severity*: `warning`
- *Action*: The AdmissionController part is down of the VPA. Debug in logs and see [upstream for more info](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/admission-controller/README.md)

### Alert Name: VPAAdmissionControllerSlow

- *Description*: `The VPA AdmissionController is slow`
- *Severity*: `warning`
- *Action*: The AdmissionController part is slow of the VPA. Requests are taking slower than 5s to complete. Debug in logs and see [upstream for more info](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/admission-controller/README.md)

### Alert Name: VPARecommenderDown

- *Description*: `The VPA Recommender is down`
- *Severity*: `warning`
- *Action*: The Recommender part is down of the VPA. Debug in logs and see [upstream for more info](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/recommender/README.md)

### Alert Name: VPARecommenderSlow

- *Description*: `The VPA Recommender is slow`
- *Severity*: `warning`
- *Action*: The Recommender part is slow of the VPA. Requests are taking slower than 5s to complete. Debug in logs and see [upstream for more info](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/recommender/README.md)

### Alert Name: VPAUpdaterDown

- *Description*: `The VPA Updater is down`
- *Severity*: `warning`
- *Action*: The Updater part is down of the VPA. Debug in logs and see [upstream for more info](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/updater/README.md)

### Alert Name: VPAUpdaterSlow

- *Description*: `The VPA Updater is slow`
- *Severity*: `warning`
- *Action*: The Updater part is slow of the VPA. Requests are taking slower than 5s to complete. Debug in logs and see [upstream for more info](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/updater/README.md)

## Other Kubernetes Runbooks and troubleshooting

- [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)
- [Recover a Broken Cluster](https://codefresh.io/Kubernetes-Tutorial/recover-broken-kubernetes-cluster/)
