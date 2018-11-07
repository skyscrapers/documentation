# Pod Disruptions

As an application owner that uses our Kubernetes cluster offering, you should be aware that disruptions can happen in the cluster, affecting your pods and the applications running on them. There's a very descriptive and informative guide in the official Kubernetes documentation about this topic, which we highly encourage you to read: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/

## TL;DR

There can be voluntary and involuntary disruptions affecting your Pods. Involuntary disruptions are mostly unavoidable hardware or system software errors. Voluntary disruptions on the other hand include cluster rolling updates, node drains and direct pod deletions, and can be avoided by creating a [`PodDisruptionBudget` object (PDB)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) for your applications.

> A PDB limits the number of pods of a replicated application that are down simultaneously from voluntary disruptions.

With correctly configured PDBs, you'll ensure that your applications will always have a minimum number of Pods running when we do a rolling upgrade of your cluster, even if all replicas are running on the same node. And the same applies to the rest of voluntary disruption scenarios.

Note that a PDB is completely different from a [Deployment strategy (`.spec.strategy`)](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy).

## Voluntary disruptions for your cluster

From a cluster administrator perspective, your cluster is susceptible to the following voluntary disruption scenarios described in the [Kubernetes documentation mentioned above](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#voluntary-and-involuntary-disruptions), and we highly recommend you to implement `PodDisruptionBudgets` so your applications are prepared for those scenarios:

- draining a node for repair or upgrade (e.g. cluster rolling upgrades)
- draining a node from a cluster to scale the cluster down (e.g. cluster autoscaling)

Additionally, from the application owner perspective (you), there might also be other types of voluntary disruptions. Although take into account that **these are not limited** by a `PodDisruptionBudget`:

- deleting the Deployment or other controller that manages the Pod
- updating a Deployment's Pod template causing a restart
- directly deleting a Pod (e.g. by accident)

> Pods which are deleted or unavailable due to a rolling upgrade to an application do count against the disruption budget, but controllers (like deployment and stateful-set) are not limited by PDBs when doing rolling upgrades

## Related topics

- [Cluster balancing](./cluster_balancing.md)
