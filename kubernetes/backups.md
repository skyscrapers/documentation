# Backups

Kubernetes clusters are backed up through [Heptio Velero](https://velero.io/). As default schedule, backups are taken each night (0:00 UTC) and are retained for 14 days, however these are configurable. Backups include:

- All defined cluster resources, encrypted and uploaded to S3
- Snapshots of EBS-backed Persistent Volumes

You can interact with your backups by installing the [`velero`](https://github.com/heptio/velero/releases) command. Some commands to get you started (note our default schedule/backups can be found in the `infrastructure` namespace):

- Configure client to use velero in the `infrastructure` namespace: `velero client config set namespace=infrastructure`
- List backup schedules: `velero schedule get`
- List available backups: `velero backup get`
- Create a manual backup: `velero backup create <mybackup>`
- Describe a backup: `velero backup describe <mybackup>`
- Get logs of a backup: `velero backup logs <mybackup>`
- Restore a complete backup: `velero restore create --from-backup <mybackup>`
- Restore a single namespace: `velero restore create --from-backup <mybackup> --include-namespaces <mynamespace>`

For more information, please consult the upstream documentation: <https://velero.io/docs>
