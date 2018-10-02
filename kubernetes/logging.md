# Kubernetes logging

We use the following components for our logging stack:
- [Fluentd](https://www.fluentd.org/)
- [AWS Cloudwatch logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
- [AWS Elasticsearch](https://aws.amazon.com/elasticsearch-service/)
- [Kibana](https://www.elastic.co/products/kibana)

## Architecture

Logs are picked up by a Fluentd `DaemonSet` that is running on every Kubernetes node. We have a default config to pick up all container loges and ship those to CloudWatch logs. We use CloudWatch logs as a buffer and archiving system. On CloudWatch logs we have a subscription stream active that streams the logs to AWS Elasticsearch with an Lambda function. Once they are in AWS Elasticsearch, we can access the logs with Kibana. You'll find your Kibana address in the `README.md` file of your GitHub repo, it'll be something like `https://kibana.staging.yourcompanyname.com` and `https://kibana.production.yourcompanyname.com`.

![Architecture](images/logging_k8s.png "Architecture")
