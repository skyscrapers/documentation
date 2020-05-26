# Network Policies

We use [Calico](https://www.projectcalico.org/) on our EKS setups as a network policy engine.

By default, Pods are non-isolated and thus accept traffic from any source. By specifying [NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) you can isolate Pods from each other and thus have more fine-grained K8s networking control.

- [K8s Network Policies documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Get started with Kubernetes network policy](https://docs.projectcalico.org/security/kubernetes-network-policy)
- [Kubernetes policy, demo](https://docs.projectcalico.org/security/tutorials/kubernetes-policy-demo/kubernetes-demo)
