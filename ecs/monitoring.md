# ECS monitoring

We use Prometheus for monitoring our ECS clusters.
We also provide you with Grafana in order to create your own dashboard with the overview of the system.

The documentation for Prometheus-specific topics can be found [here](../tools/prometheus.md).

We defined a terraform module to deploy Prometheus on ECS.
The module deploys Prometheus with AlertManager and Grafana. It uses EFS to share the configuration between the cluster nodes.


## ECS application monitoring

The basic setup allows to scrape for node-exporter metrics and cloudwatch metrics and we can add as many custom metrics as needed.

Currently we can access prometheus at the URL "https://prometheus.your.domain:9090".
For Grafana you can use the following url: "https://grafana.your.domain:9090".
The access to that port is IP-restricted so you will need to communicate a static IP we can enable to access the prometheus/grafana dashboard or implement a VPN connection to the VPC.
