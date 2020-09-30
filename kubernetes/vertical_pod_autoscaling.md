# Vertical-pod-autoscaling

[Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) frees the users from necessity of setting up-to-date resource limits and requests for the containers in their pods. When configured, it will set the requests automatically based on usage and thus allow proper scheduling onto nodes so that appropriate resource amount is available for each pod. It will also maintain ratios between limits and requests that were specified in initial containers configuration.

It can both down-scale pods that are over-requesting resources, and also up-scale pods that are under-requesting resources based on their usage over time.

- [Vertical-pod-autoscaling](#vertical-pod-autoscaling)
  - [Configuration](#configuration)

## Configuration

Autoscaling is configured with a
[Custom Resource Definition object](https://kubernetes.io/docs/concepts/api-extension/custom-resources/)
called [VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/apis/autoscaling.k8s.io/v1/types.go).
It allows to specify which pods should be vertically autoscaled as well as if/how the
resource recommendations are applied and what the boundries are.

``` yaml
---
apiVersion: "autoscaling.k8s.io/v1beta2"
kind: VerticalPodAutoscaler
metadata:
  name: example-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-example-deployment
  resourcePolicy:
    containerPolicies:
      - containerName: "application"
        minAllowed:
          cpu: 10m
          memory: 50Mi
        maxAllowed:
          cpu: 100m
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
```

In this example we allow the `application` container in the `my-example-deployment` deployment to scale between the `minAllowed` and the `maxAllowed` boundries.

**Note:** we allow single replica deployments to be autoscaled by the VPA but do note that the containers needs to be restarted when the VPA makes any adjustments to the container and that that causes a downtime.
**Note:** Vertical Pod Autoscaler **should not be used with the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) (HPA) on CPU or memory** at this moment.
