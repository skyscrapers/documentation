# Networking

- [Network Policies](./network_policies.md)
- [VPC CNI](#vpc-cni)

## VPC CNI

### Overview

The VPC CNI (Container Network Interface) plugin is the default networking plugin for Amazon EKS clusters. The CNI plugin allows Kubernetes Pods to have the same IP address as they do on the VPC network. More specifically, all containers inside the Pod share a network namespace, and they can communicate with each-other using local ports.

### Configuration Parameters

The following parameters for the VPC CNI plugin can be set through the cluster-definition file:

- network_policy_enabled (default enabled): whether to enable the VPC CNI NetworkPolicy agent (NPAgent)
- readiness_probe_timeout_seconds and liveness_probe_timeout_seconds (default 10): timeout in seconds for the VPC CNI probes.

#### Best Practices

#### Probe and Resource Settings

For EKS 1.20 and later clusters, AWS advises increasing the liveness and readiness probe timeout values (default `timeoutSeconds: 10`) for the `aws-node` DaemonSet. This helps prevent probe failures that can cause Pods to become stuck in a `ContainerCreating` state, especially in data-intensive and batch-processing clusters. High CPU usage may lead to `aws-node` probe health failures, resulting in unfulfilled Pod CPU requests.

In addition to modifying the probe timeout, ensure that the CPU resource requests (default CPU: `25m`) for `aws-node` are correctly configured. Only update these settings if your node is experiencing issues; otherwise, the defaults are sufficient for most workloads.

## References

- [Amazon EKS VPC CNI userguide](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)
- [Amazon EKS VPC CNI Best practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html)
- [GitHub: amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s)
