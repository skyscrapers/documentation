# Pod Autoscaling

## Horizontal Pod Autoscaling

We provide [KEDA (Kubernetes Event-driven Autoscaling)](https://keda.sh/) for scaling Pods horizontally, based on metrics like resource usage, queue length, etc.

From the upstream project:

> KEDA is a [Kubernetes](<https://kubernetes.io/)-based> Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.
>
> KEDA is a single-purpose and lightweight component that can be added into any Kubernetes cluster. KEDA works alongside standard Kubernetes components like the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) and can extend functionality without overwriting or duplication. With KEDA you can explicitly map the apps you want to use event-driven scale, with other apps continuing to function. This makes KEDA a flexible and safe option to run alongside any number of any other Kubernetes applications or frameworks.

### Configuration

Autoscaling is configured via objects
called [ScaledObject (for Deployments, StatefulSets and Custom Resources)](https://keda.sh/docs/2.6/concepts/scaling-deployments/) and [ScaledJob (for Jobs)](https://keda.sh/docs/2.6/concepts/scaling-jobs/).

We recommend to check out the linked upstream documentation, in addition to the [documentation of a specific Scaler](https://keda.sh/docs/2.6/scalers/). We provide some common examples below to get you started.

CPU scaling example ([docs](https://keda.sh/docs/2.6/scalers/cpu/)):

``` yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myapp
  namespace: mynamespace
spec:
  scaleTargetRef:
    kind: Deployment
    name: myapp
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: cpu
      metadata:
        type: Utilization
        value: "60"
```

SQS scaling example ([docs](https://keda.sh/docs/2.6/scalers/aws-sqs/)):

``` yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myapp
  namespace: mynamespace
spec:
  scaleTargetRef:
    kind: Deployment
    name: myapp
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://mysqsqueueurl
        queueLength: "50"   # Number of messages per Pod
        awsRegion: eu-west-1
        identityOwner: operator   # Use IAM permissions from the Keda operator
```

And many more options, [check out all scalers](https://keda.sh/docs/2.6/scalers/)! Make sure to contact us for specific usecases, eg. to help out setting IAM permissions correctly.

**Important**: KEDA uses the [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) under the hood. Make sure you **don't** define both HPA and Keda resources for scaling the same workload as they will compete with each other and this will result in odd scaling behavior. Instead replace the HPA with the Keda [CPU](https://keda.sh/docs/latest/scalers/cpu/) and [Memory](https://keda.sh/docs/latest/scalers/memory/) scalers.

## Vertical Pod Autoscaling

[Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) frees the users from the necessity of setting up-to-date resource limits and requests for the containers in their pods. When configured, it will set the requests automatically based on usage and thus allow proper scheduling onto nodes so that the appropriate resource amount is available for each pod. It will also maintain the ratios between limits and requests that were specified in the initial containers configuration.

It can both down-scale pods that are over-requesting resources, and also up-scale pods that are under-requesting resources based on their usage over time.

### Configuration

Autoscaling is configured with a
[Custom Resource Definition object](https://kubernetes.io/docs/concepts/api-extension/custom-resources/)
called [VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/apis/autoscaling.k8s.io/v1/types.go).
It allows to specify which pods should be vertically autoscaled as well as if/how the
resource recommendations are applied and what the boundries are.

``` yaml
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

**Note:** we allow single replica deployments to be autoscaled by the VPA but do note that the containers need to be restarted when the VPA makes any adjustments to the container and that that causes a downtime.
**Note:** Vertical Pod Autoscaler **should not be used with the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) (HPA) on CPU or memory** at this moment.
