<!-- markdownlint-disable MD024 MD034 -->
# Pods

Kubernetes, an open-source container orchestration platform, revolutionizes the deployment, scaling, and management of containerized applications. At the heart of Kubernetes is the concept of a Pod, the smallest and simplest unit in the Kubernetes object model.

- [Pods](#pods)
  - [What is a Pod?](#what-is-a-pod)
  - [Key Characteristics of Pods](#key-characteristics-of-pods)
  - [Use Cases for Pods](#use-cases-for-pods)
  - [Pod Lifecycle](#pod-lifecycle)
  - [Managing Pods](#managing-pods)
  - [Resource requests and limits](#resource-requests-and-limits)
    - [Importance of Resource Requests and Limits](#importance-of-resource-requests-and-limits)
    - [How to Set Resource Requests and Limits](#how-to-set-resource-requests-and-limits)
    - [Best Practices for Setting Resource Requests and Limits](#best-practices-for-setting-resource-requests-and-limits)
  - [Scheduling and reliability](#scheduling-and-reliability)
    - [Autoscaling](#autoscaling)
      - [Horizontal Pod Autoscaling](#horizontal-pod-autoscaling)
        - [Configuration](#configuration)
      - [Vertical Pod Autoscaling](#vertical-pod-autoscaling)
        - [Configuration](#configuration-1)
    - [Pod Disruption Budgets](#pod-disruption-budgets)
      - [TL;DR](#tldr)
      - [Best Practices](#best-practices)
    - [Pod (Anti-)Affinity](#pod-anti-affinity)
    - [Quality of Service Classes](#quality-of-service-classes)
    - [Probes (Liveness, Readiness, and Startup)](#probes-liveness-readiness-and-startup)
  - [Conclusion](#conclusion)

## What is a Pod?

A Pod represents a single instance of a running process in a Kubernetes cluster. It is the smallest deployable unit that can be created, scheduled, and managed by Kubernetes. A Pod encapsulates one or more containers (such as Docker containers), storage resources, a unique network IP, and options that govern how the containers should run.

## Key Characteristics of Pods

1. **Single or Multiple Containers**: A Pod can contain one or multiple containers. When a Pod contains multiple containers, they are tightly coupled, sharing the same resources and running in the same execution environment. These containers can share storage volumes and the network namespace, allowing them to communicate with each other using `localhost`.

2. **Shared Network**: All containers in a Pod share the same IP address and port space, making inter-container communication straightforward. This shared network model means that within a Pod, containers can easily communicate with each other over `localhost`.

3. **Storage**: Pods can specify a set of shared storage volumes that the containers within the Pod can access. This enables data sharing and persistence across container restarts.

4. **Lifecycle and Restart Policies**: Pods include specifications for how containers should be restarted, managed, and terminated. Kubernetes manages the lifecycle of Pods, ensuring that the desired state is maintained, such as restarting containers if they fail.

## Use Cases for Pods

1. **Single Container Pod**: This is the most common use case, where each Pod contains a single container. This model aligns closely with the traditional way of deploying applications as individual containers.

2. **Multi-Container Pod**: Sometimes, it is beneficial to run multiple containers together within a single Pod. Examples include a web server container paired with a logging sidecar container, or a primary application container coupled with a helper container that provisions data.

## Pod Lifecycle

1. **Pending**: The Pod has been accepted by the Kubernetes system but one or more of the container images has not been created yet.

2. **Running**: The Pod has been bound to a node, and all of the containers have been created. At least one container is still running or is in the process of starting or restarting.

3. **Succeeded**: All containers in the Pod have terminated successfully, and will not be restarted.

4. **Failed**: All containers in the Pod have terminated, and at least one container has terminated in failure.

5. **Unknown**: The state of the Pod could not be obtained, typically due to an issue in communicating with the host of the Pod.

## Managing Pods

Kubernetes provides several abstractions for managing Pods:

1. **ReplicationController**: Ensures that a specified number of Pod replicas are running at any given time.

2. **ReplicaSet**: A newer version of ReplicationController with additional features, primarily used by Deployments.

3. **Deployment**: Provides declarative updates to Pods and ReplicaSets, allowing for rolling updates and rollbacks.

4. **StatefulSet**: Manages stateful applications, guaranteeing the ordering and uniqueness of Pods.

5. **DaemonSet**: Ensures that a copy of a Pod runs on all (or some) nodes.

6. **Job**: Creates one or more Pods and ensures that a specified number of them successfully terminate.

## Resource requests and limits

Resource requests and limits are essential tools for managing the CPU and memory resources consumed by pods in a Kubernetes cluster. Properly setting these values ensures that your applications run efficiently, remain stable, and coexist harmoniously with other workloads.

### Importance of Resource Requests and Limits

1. **Efficient Resource Allocation**: Kubernetes uses resource requests to schedule pods on nodes that have sufficient available resources. This prevents overcommitment and ensures that your applications have the resources they need to run smoothly.
2. **Preventing Resource Exhaustion**: By setting limits, you can prevent a single pod from consuming all available resources on a node, which could degrade the performance of other applications.
3. **Quality of Service (QoS) Classes**: Kubernetes assigns QoS classes based on the resource requests and limits defined for a pod. These classes help Kubernetes make better decisions during resource contention.

### How to Set Resource Requests and Limits

Resource requests and limits are defined in the pod specification for each container within a pod. Here's an example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: my-container
    image: my-image
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
```

In this example:

- **requests**: Specifies the minimum amount of CPU and memory required for the container.
- **limits**: Specifies the maximum amount of CPU and memory the container is allowed to use.

### Best Practices for Setting Resource Requests and Limits

1. **Understand Your Application's Resource Needs**: Monitor your application's CPU and memory usage to determine appropriate request and limit values. Tools like Prometheus and Grafana can help with this.

2. **Set cpu requests**. We don't recommend setting CPU limits

3. **Set memory requests and limits** same as requests unless there's a very good reason to handle certain peaks

4. **Set Reasonable Defaults**: Define reasonable default values for requests and limits based on typical application behavior. Avoid setting them too high or too low.

5. **Regularly Review and Adjust**: Periodically review the resource usage of your pods and adjust the requests and limits as needed to adapt to changing workloads.

6. **Avoid Overcommitting Resources**: Be cautious with setting limits much higher than requests. Overcommitting can lead to resource contention and degraded performance.

7. **Consider Using [Autoscaling](#autoscaling)**

By regularly monitoring and adjusting the resource requests and limits, you can ensure that your applications run efficiently and remain responsive under varying loads.

## Scheduling and reliability

### Autoscaling

#### Horizontal Pod Autoscaling

We provide [KEDA (Kubernetes Event-driven Autoscaling)](https://keda.sh/) for scaling Pods horizontally, based on metrics like resource usage, queue length, etc.

From the upstream project:

> KEDA is a [Kubernetes](<https://kubernetes.io/)-based> Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.
>
> KEDA is a single-purpose and lightweight component that can be added into any Kubernetes cluster. KEDA works alongside standard Kubernetes components like the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) and can extend functionality without overwriting or duplication. With KEDA you can explicitly map the apps you want to use event-driven scale, with other apps continuing to function. This makes KEDA a flexible and safe option to run alongside any number of any other Kubernetes applications or frameworks.

##### Configuration

Autoscaling is configured via objects
called [ScaledObject (for Deployments, StatefulSets and Custom Resources)](https://keda.sh/docs/2.11/concepts/scaling-deployments/) and [ScaledJob (for Jobs)](https://keda.sh/docs/2.11/concepts/scaling-jobs/).

We recommend to check out the linked upstream documentation, in addition to the [documentation of a specific Scaler](https://keda.sh/docs/2.11/scalers/). We provide some common examples below to get you started.

CPU scaling example ([docs](https://keda.sh/docs/2.11/scalers/cpu/)):

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
      metricType: Utilization
      metadata:
        value: "60"
```

SQS scaling example ([docs](https://keda.sh/docs/2.11/scalers/aws-sqs/)), make sure to contact us first to enable the necesseary IAM permissions:

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

#### Vertical Pod Autoscaling

[Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) frees the users from the necessity of setting up-to-date resource limits and requests for the containers in their pods. When configured, it will set the requests automatically based on usage and thus allow proper scheduling onto nodes so that the appropriate resource amount is available for each pod. It will also maintain the ratios between limits and requests that were specified in the initial containers configuration.

It can both down-scale pods that are over-requesting resources, and also up-scale pods that are under-requesting resources based on their usage over time.

##### Configuration

Autoscaling is configured with a
[Custom Resource Definition object](https://kubernetes.io/docs/concepts/api-extension/custom-resources/)
called [VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/apis/autoscaling.k8s.io/v1/types.go).
It allows to specify which pods should be vertically autoscaled as well as if/how the
resource recommendations are applied and what the boundries are.

``` yaml
apiVersion: "autoscaling.k8s.io/v1"
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

### Pod Disruption Budgets

As an application owner that uses our Kubernetes cluster offering, you should be aware that disruptions can happen in the cluster, affecting your pods and the applications running on them. There's a very descriptive and informative guide in the official Kubernetes documentation about this topic, which we highly encourage you to read: <https://kubernetes.io/docs/concepts/workloads/pods/disruptions/>

#### TL;DR

There can be voluntary and involuntary disruptions affecting your Pods. Involuntary disruptions are mostly unavoidable hardware or system software errors. Voluntary disruptions on the other hand include cluster rolling updates, node drains and direct pod deletions, and can be avoided by creating a [`PodDisruptionBudget` object (PDB)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) for your applications.

> A PDB limits the number of pods of a replicated application that are down simultaneously from voluntary disruptions.

With correctly configured PDBs, you'll ensure that your applications will always have a minimum number of Pods running when we do a rolling upgrade of your cluster, even if all replicas are running on the same node. And the same applies to the rest of voluntary disruption scenarios.

Note that a PDB is completely different from a [Deployment strategy (`.spec.strategy`)](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy).

#### Best Practices

1. **Define Clear Availability Requirements**: Specify the minimum number or percentage of pods that must be available during disruptions to avoid service downtime.

    ```yaml
    apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: my-service-pdb
    spec:
      minAvailable: 1
      selector:
        matchLabels:
          app: my-service
    ```

2. **Do not set PDBs on single repica workloads**: there should not be a PDB set as this will block voluntary rollouts of that workload or node where this pod is running on.

3. **Use `minAvailable` or `maxUnavailable`**: Depending on the nature of your service, use either `minAvailable` (minimum number of pods that must be available) or `maxUnavailable` (maximum number of pods that can be unavailable).

4. **Regularly Update PDBs**: Ensure that your PDBs are up-to-date with the current scaling of your application to avoid conflicts during maintenance or scaling operations.

5. **Monitor PDBs**: Regularly monitor the status of your PDBs to ensure they are being respected and are not causing issues with deployments or maintenance activities.

More best practices can be found in the AWS best practices guide:

- <https://aws.github.io/aws-eks-best-practices/reliability/docs/application/>
- <https://aws.github.io/aws-eks-best-practices/karpenter/#scheduling-pods>

### Pod (Anti-)Affinity

Pod affinity and anti-affinity rules allow you to control the placement of pods relative to other pods, either to co-locate them (affinity) or to separate them (anti-affinity).

**Best Practices:**

1. **Use Affinity for Localized Workloads**: Co-locate pods that communicate frequently to reduce network latency and increase performance.

    ```yaml
    affinity:
      podAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-app
          topologyKey: "kubernetes.io/hostname"
    ```

2. **Use Anti-Affinity for High Availability**: Distribute pods across nodes or zones to increase fault tolerance and availability.

    ```yaml
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-app
          topologyKey: "topology.kubernetes.io/zone"
    ```

3. **Soft and Hard Rules**: Use `requiredDuringSchedulingIgnoredDuringExecution` for hard constraints and `preferredDuringSchedulingIgnoredDuringExecution` for soft constraints to balance between availability and flexibility.

4. **Consider Topology Keys**: Use appropriate topology keys such as `kubernetes.io/hostname` for node-level policies or `failure-domain.beta.kubernetes.io/zone` for zone-level policies.

### Quality of Service Classes

Quality of Service (QoS) classes in Kubernetes help ensure that critical applications get the resources they need by categorizing pods based on their resource requests and limits.

**Best Practices:**

1. **Define Resource Requests and Limits**: Specify requests and limits for CPU and memory to ensure proper scheduling and resource allocation.

  ```yaml
  spec:
    containers:
    - name: my-container
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
  ```

  > [!NOTE]
  > Generally we don't recommend setting CPU limits, only requests. However is required if you want to enforce the Guaranteed QoS class.

1. **Understand QoS Classes**:
   - **Guaranteed**: Set both requests and limits for all containers, making the pod less likely to be evicted.
   - **Burstable**: Set requests lower than limits, providing a balance between guaranteed and best-effort resources.
   - **Best-Effort**: Do not set requests or limits, making the pod the first to be evicted under resource pressure.

2. **Monitor Resource Usage**: Regularly monitor the actual resource usage of your pods to adjust requests and limits as needed.

3. **Optimize for Performance and Cost**: Use QoS classes to balance performance needs with cost efficiency by appropriately setting requests and limits.

### Probes (Liveness, Readiness, and Startup)

Probes are crucial for ensuring that your applications are running correctly and are ready to serve traffic.

**Best Practices:**

1. **Use Liveness Probes to Detect and Restart Unresponsive Pods**:

    ```yaml
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
    ```

2. **Use Readiness Probes to Ensure Pods are Ready to Serve Traffic**:

    ```yaml
    readinessProbe:
      httpGet:
        path: /readiness
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
    ```

3. **Use Startup Probes for Applications with Long Initialization**:

    ```yaml
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    ```

4. **Adjust Probe Timing Appropriately**: Set `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, and `failureThreshold` based on your application's behavior to avoid false positives.

5. **Test Probes Thoroughly**: Ensure your probes accurately reflect the health and readiness of your application by testing in staging environments before deploying to production.

6. **Monitor Probe Results**: Continuously monitor the outcomes of probes to identify and rectify issues promptly, ensuring high availability and reliability of your services.

By following these best practices, you can effectively manage Kubernetes Pods, ensuring high availability, efficient resource utilization, and robust application performance.

## Conclusion

Pods are a fundamental concept in Kubernetes, encapsulating the application containers, storage, and network resources needed to run an application. Understanding Pods is essential for leveraging Kubernetes to deploy, manage, and scale containerized applications effectively. Through Pods, Kubernetes provides a powerful yet flexible model for orchestrating the lifecycle of application components in a clustered environment.
