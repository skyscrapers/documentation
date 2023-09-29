# Network Policies

We enable support [Kubernetes NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) on all clusters.

By default, Pods are non-isolated and thus accept traffic from any source. By specifying [NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) you can isolate Pods from each other and thus have more fine-grained K8s networking control.

- [K8s Network Policies documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
