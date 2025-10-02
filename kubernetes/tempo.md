# Grafana Tempo

This guide explains how to use Grafana Tempo for distributed tracing in your applications running on Kubernetes. It’s written for Skyscrapers customers who want to use tracing.

## Overview

Grafana Tempo is a highly scalable, cost-effective tracing backend that integrates seamlessly with Grafana. It supports the OpenTelemetry Protocol (OTLP), making it easy to instrument your applications.

## How Tempo is Set Up (High Level)

The Tempo setup in your Kubernetes cluster consists of:

- [Alloy as Collector](https://grafana.com/docs/alloy/latest/): Applications send traces via the OpenTelemetry Protocol (OTLP) to the Alloy endpoint on the cluster.
- [Tempo backend](https://grafana.com/docs/tempo/latest/): deployed as a single-tenant service inside your Kubernetes cluster with S3 storage.
- Grafana integration: Traces can be visualized in Grafana and linked with logs and metrics.

The setup can be configured by enabling the `observability.tempo.enabled` flag in your cluster definition file.

## Application Instrumentation

### Demo Application

Want to see tracing in action? Deploy the [Mythical Beasts](https://github.com/grafana/intro-to-mltp) demo application in your cluster. It comes pre-instrumented with OpenTelemetry and is a great way to explore tracing.

You can do this by following the instructions in the [Tempo documentation here](https://grafana.com/docs/tempo/latest/setup/set-up-test-app/#test-your-configuration-using-the-intro-to-mltp-application).

> [!NOTE]
> Adjust the tracing endpoint in the `mythical-beasts-deployment.yaml` file to point to your Alloy service: `alloy.observability.svc.cluster.local`.

### Install OpenTelemetry SDK/Agent

Choose the library for your language/framework:

- Go: [opentelemetry-go](https://github.com/open-telemetry/opentelemetry-go)
- Java: [opentelemetry-java](https://github.com/open-telemetry/opentelemetry-java) or [Java auto-instrumentation agent](https://github.com/open-telemetry/opentelemetry-java-instrumentation)
- Python: [opentelemetry-python](https://github.com/open-telemetry/opentelemetry-python)
- Node.js: [opentelemetry-js](https://github.com/open-telemetry/opentelemetry-js)

### Configure Exporter

Each SDK/agent needs to know **where to send traces**:

```yaml
OTEL_EXPORTER_OTLP_ENDPOINT=http://alloy.observability.svc.cluster.local:4317
OTEL_EXPORTER_OTLP_TRACES_INSECURE=true
OTEL_RESOURCE_ATTRIBUTES=service.name=<my-app>,service.namespace=<my-namespace> # optional
```

You can set these as **environment variables** in your Kubernetes `Deployment` manifest.

Example (Kubernetes snippet):

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://alloy.observability.svc.cluster.local:4317"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "service.name=<my-app>,service.namespace=<my-namespace>"
```

### Auto vs Manual Instrumentation

👉 **Recommended approach**: Start with **auto-instrumentation** to get quick visibility. Then, gradually add **manual instrumentation** where you need deeper insights.

#### Auto-Instrumentation

What it is: Automatically hooks into common libraries and frameworks (HTTP servers, gRPC, database clients, message queues, etc.).
When to use: Great for getting started quickly or when you want broad coverage with minimal code changes.

How to enable:

- Java: Run your app with the OpenTelemetry Java agent: `java -javaagent:opentelemetry-javaagent.jar -jar app.jar`
- Python: Use the CLI wrapper: `opentelemetry-instrument python app.py`
- Node.js: Load the instrumentation before your app starts: `node -r @opentelemetry/auto-instrumentations-node app.js`

Pros: Fast to adopt, no or minimal code changes.
Cons: Limited control over which spans are created and what metadata is added.

#### Manual Instrumentation

What it is: Adding explicit tracing code in your application.
When to use: Useful when you need detailed insights into specific functions, business logic, or performance-sensitive areas.
How to add (example in Go):

```go
import "go.opentelemetry.io/otel"
import "go.opentelemetry.io/otel/trace"

var tracer = otel.Tracer("my-service")

func handleRequest(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "handleRequest")
    defer span.End()

    // Your application logic here
}
```

Pros: Full control over what gets traced, ability to add meaningful custom attributes.
Cons: Requires code changes and ongoing maintenance.

## Viewing Traces in Grafana

1. Open **Grafana** (provided by Skyscrapers).
2. Go to **Explore** and select Tempo from the dropdown.
3. Search for recent traces.
4. Drill down into spans to see:

- Latency breakdown
- Parent/child relationships
- Linked logs and metrics

![Grafana Tempo View](images/grafana-tempo.pngg)

## Resources

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Grafana Tempo Docs](https://grafana.com/docs/tempo/latest/)
