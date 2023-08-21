## Backups

### Automated Backups for AWS RDS

AWS RDS provides automated backups that run every 5 minutes. These automated backups allow for quick point-in-time recovery and are an essential part of data protection.

- **Backup Frequency**: Automated backups are taken every 5 minutes.
- **Retention Period**: By default, AWS RDS retains automated backups for **14 days**.
- **Configurability**: You can configure the retention period according to your specific requirements.

AWS RDS automated backups are a reliable way to ensure data availability and recoverability in case of unexpected incidents.

### Retention Period

In general, we use the following retention periods by default. However, most of these schedules can be configured.

- AWS RDS: Daily snapshots, which also allows for point-in-time recovery. Our default retention is **14 days**. Configurable
- AWS OpenSearch (previously known as ElasticSearch) Service:
  - AWS-provided hourly snapshot, with a retention of **14 days**. Not configurable
  - Skyscrapers-provided snapshot every 6 hours, with a **14 day retention**. Snapshots are stored on an encrypted S3 bucket. Configurable
- AWS ElastiCache for Redis: by default snapshots are **disabled** but is configurable
- Kubernetes state and Statefulset volumes are backed up daily through Velero with a default **14 day** retention). Further documentation: [/kubernets/backups.md](/kubernetes/backups.md). Configurable
  - Kubernetes state: all objects are backed up to S3 (encrypted)
  - EBS volumes: uses AWS snapshots, encrypted if the original volume is encrypted (so default **yes**, if you use the `gp2-encrypted` Storage Class is used for your PVCs)
- MongoDB: daily backup with **14 day** retention. Configurable
- AWS Redshift: default snapshot schedule, with **1 day** retention (configurable), as [specified in the AWS docs](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-snapshots.html#about-automated-snapshots)

