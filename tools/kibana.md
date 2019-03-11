# Elasticsearch Kibana

## Logging in

To login you can browse to https://kibana.<<clustername>>. The login is managed like our other tools with Dex.

## How to use kibana

### Querying data

all data from the cluster is sent to the `cwl-*` source. in here you can query your data.

Quick queries can be executed in the Discover pane of the Kibana UI. A small introduction to querying the data can be found [here](https://www.elastic.co/guide/en/kibana/current/discover.html)

#### Frequently used queries

- get all logs from nginx-ingress-controller: kubernetes.container_name: nginx-ingress-controller

### Graphs and dashboards

#### Building visualizations

If you want to create a single graph on some data metrics a visualization is what you need. How you can set this up can be found [here](https://www.elastic.co/guide/en/kibana/current/tutorial-visualizing.html). If you want to create multiple graphs on the same window you need to create a dashboard.

#### Building dashboards

For this we want to refer to [the elastic documentation](https://www.elastic.co/guide/en/kibana/current/tutorial-build-dashboard.html).
