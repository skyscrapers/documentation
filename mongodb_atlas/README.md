# MongoDB Atlas

We standardise on [MongoDB Atlas](https://www.mongodb.com/atlas) service four our customer's MongoDB clusters. Atlas is a fully-managed service to deploy, manage and monitor MongoDB clusters. It's multi-cloud, so you can choose from various cloud providers to deploy your clusters to, including AWS.

Here are some lessons learned from past experience working with Altas for our customers:

## Data migration

Atlas offers a managed [Live Migration Service](https://www.mongodb.com/cloud/atlas/migrate) to import data into an Atlas MongoDB cluster from a self-managed community cluster, which is likely the scenario customers face when moving into Atlas. This service comes with a few limitations though, and some of them are not easy to overcome. Those limitations are listed in their documentation [here](https://docs.atlas.mongodb.com/import/live-import/#restrictions).

One of the restrictions that might not be easily deduced from the documentation is that the Live Migration Service only supports migrating from Replica Sets. If your source cluster is a standalone server, you'll first need to convert that into a Replica Set in order to use the service.

Although the most remarkable of the restrictions would be the one about not supporting private clusters. Atlas Live Migration Service only supports migrating data from public MongoDB clusters, one where all the Replica Set members are publicly reachable from the internet, and where comunication between these Replica Set members is also performed over the internet. The Atlas service performs a discovery process at the beginning of the migration, and uses the hostnames / IP addresses configured in the source Replica Set to target each of the servers to pull the data from. So if your source cluster is configured to communicate through a private network (e.g. VPC), the Live Migration Service won't work, even if the servers themselves are publicly reachable.

To overcome this limitation, you'll need to use [`mongomirror`](https://docs.atlas.mongodb.com/import/mongomirror/), which is actually the tool that Atlas Live Migration Service uses under the hood. `mongomirror` comes with its own [set of prerequisites](https://docs.atlas.mongodb.com/import/mongomirror/#prerequisites), but you can run it from a private replica set node without much trouble. It's quite simple to setup and run, and it doesn't add much overhead on the running source MongoDB cluster, so you can safely run it on a live cluster. `mongomirror` will perform the same discovery process of your source replica set, and will start migrating data from the primary node of the cluster, so to avoid extra transfer costs, it's best to run it from the same primary node instance.

## Snapshots export to S3

Atlas offers the option of [exporting cluster snapshots into an S3 bucket](https://docs.atlas.mongodb.com/backup/cloud-backup/export/), in the extended JSON format. It also offers the option of setting up an export policy for automatic export of snapshots. The problem is that this export policy can only be configured to perform an export job per month, which is not enough in some cases.

Another limitation of this service is that it can only be [managed through the Atlas API](https://docs.atlas.mongodb.com/backup/cloud-backup/export/#export-management). There's no UI option in the web console, nor Terraform resource to manage it, so it's not that straight-forward to set up.

In order to set up these snapshot exports, you need to do the following:

- create an S3 bucket where to put the exported data
- [configure IAM access for Atlas](https://docs.atlas.mongodb.com/security/set-up-unified-aws-access/) to access your S3 bucket
  - the IAM role needs at list the `s3:PutObject` and `s3:getBucketLocation` actions on the S3 bucket
- [create an "export bucket" in Atlas](https://docs.atlas.mongodb.com/reference/api/cloud-backup/export/create-one-export-bucket). This is an Atlas API call to configure it to access the bucket you created in AWS

After this is done, you can either configure the cluster's backup policy to automatically export snapshots on a montly basis or [manually trigger export jobs](https://docs.atlas.mongodb.com/reference/api/cloud-backup/export/create-one-export-job) at a custom frequency. To set up the backup policy, you'll need to make yet another [API call to Atlas](https://docs.atlas.mongodb.com/reference/api/cloud-backup/schedule/modify-one-schedule/), which **must originate from an IP address in the organization's API access list**.
