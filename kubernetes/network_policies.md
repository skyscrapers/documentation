# Network Policies

We enable support [Kubernetes NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) on all clusters.

## Introduction

This documentation provides detailed instructions on how to manage and deploy Kubernetes Network Policies using Helm. Kubernetes Network Policies are used to control the traffic flow between pods and/or between pods and network endpoints. They define how groups of pods are allowed to communicate with each other and other network endpoints.

By default, Pods are non-isolated and thus accept traffic from any source. By specifying [NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) you can isolate Pods from each other and thus have more fine-grained K8s networking control.

## Prerequisites

- A running Kubernetes cluster
- Helm installed and configured
- `kubectl` installed and configured
- Basic understanding of Kubernetes Network Policies

### Example polciy

Here is an example of a Kubernetes Network Policy YAML file. This policy restricts traffic so that only pods with a specific label can communicate with the selected pods.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: allowed-app
    ports:
    - protocol: TCP
      port: 80
```

#### Explanation

- **apiVersion**: Specifies the API version.
- **kind**: Indicates that this is a NetworkPolicy resource.
- **metadata**: Contains the name and namespace of the NetworkPolicy.
- **spec**: Defines the desired behavior of the NetworkPolicy.
  - **podSelector**: Selects the pods to which this policy applies. In this case, it targets pods labeled with `app: my-app`.
  - **policyTypes**: Specifies the types of traffic to be controlled. Here, it controls ingress (incoming) traffic.
  - **ingress**: Lists the ingress rules.
    - **from**: Defines the sources that are allowed to access the selected pods. In this example, only pods labeled with `app: allowed-app` are permitted.
    - **ports**: Specifies the allowed ports. Here, only TCP traffic on port 80 is allowed.

This Network Policy ensures that only pods with the label `app: allowed-app` can send TCP traffic on port 80 to the pods labeled `app: my-app` in the `default` namespace.

## Best Practices

- Use descriptive names for your Network Policies to easily identify them.
- Regularly review and update your Network Policies to ensure they meet the current security requirements.
- Use namespaces to isolate Network Policies for different environments or applications.

## Troubleshooting

### Common Issues

1. **Network Policy not applied**: Ensure that the `podSelector` and `namespace` fields are correctly set and match the pods you want to target.
2. **Incorrect policy types**: Verify that `policyTypes` includes the correct types (`Ingress` and/or `Egress`) for your use case.

### Viewing Network Policies

To view the Network Policies applied in a namespace, use the following command:

```sh
kubectl get networkpolicies -n <namespace>
```

Replace `<namespace>` with the appropriate namespace.

### Checking Network Policy Logs

Check the logs of your network plugin (e.g., Calico, Weave) to debug issues related to Network Policies.

## Conclusion

Helm makes it easier to manage and deploy Kubernetes Network Policies. By using Helm templates, you can customize and scale your Network Policies efficiently. Follow the best practices and troubleshooting tips provided in this documentation to ensure your network policies are effective and secure.

For further reading and detailed specifications, refer to the [Kubernetes Network Policy documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
