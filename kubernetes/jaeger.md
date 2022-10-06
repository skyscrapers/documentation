# Jaeger distributed tracing

The Skyscrapers Kubernetes reference solution has built-in traceability and observability features, based on [Jaeger](https://www.jaegertracing.io/) and its [Jaeger Operator](https://www.jaegertracing.io/docs/1.38/operator/). These functions can be enabled as an optional add-on. Note that as of now, this is only available for EKS clusters. AKS clusters will follow in the near future, depending on customer demand.

If you think this could be useful for your team, get in touch with us so we can enable it on your cluster(s).

## Introduction

Jaeger is an open-source observability tool for distributed tracing. It helps developers troubleshoot, debug and monitor their applications and workflows.

When enabled, one or more Jaeger instances can be deployed on a Kubernetes cluster. Normally a single Jaeger should be enough, but for those customers that run multiple environments on the same cluster and want to keep the traces separated, a different Jaeger per environment can be deployed as well. All Jaeger instances will be deployed in the `observability` namespace.

## How to use it

Your applications must be instrumented before they can send tracing data to Jaeger. Jaeger recommends using the [OpenTelemetry](https://opentelemetry.io/) instrumentation and SDKs.

After your applications are instrumented properly, you simply add the Jaeger injection annotation (`sidecar.jaegertracing.io/inject`) to your `Deployment(s)` and the Jaeger Operator will automatically inject a Jaeger agent into the application `Pod(s)`. This agent will automatically forward the application traces to the Jager instance.

The values can be either `"true"` (as string), or the Jaeger instance name, as returned by `kubectl get jaegers -n observability`. Note that `"true"` can only be used when there's exactly one Jaeger instance deployed.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    "sidecar.jaegertracing.io/inject": "true"
spec:
  ...
```

You can read more about how the auto-injection mechanism works in the [Jaeger Operator documentation](https://www.jaegertracing.io/docs/1.38/operator/#auto-injecting-jaeger-agent-sidecars).

Our managed Jaeger instances already come with an `Ingress` configured to access their UI. Check them out with `kubectl get ingresses -n observability`.

## Deployment strategies

Jaeger can be deployed in [two different strategies](https://www.jaegertracing.io/docs/1.38/operator/#deployment-strategies): `allinone` or `production`. Note that the official documentation mentions a `streaming` strategy, but that's not yet supported in our platforms.

The default `allinone` strategy combines all Jaeger components (agent, collector and query service) into a single executable running in a single `Pod`, and it uses in-memory storage, so there's no persistance. `allinone` is only intended for development / staging environments.

For production and other critical environments, the `production` strategy should be used, which deploys each Jaeger component separately, and uses Elasticsearch for persistance. By default, our platform automatically provisions an AWS Opensearch cluster (in the same VPC as the EKS cluster) for those Jaeger instances that use the `production` deployment strategy, but it's also possible to configure a custom Elasticsearch endpoint if needed.
