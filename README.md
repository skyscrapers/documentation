# Skyscrapers products & services documentation

This repository contains user-level documentation for our product & service offering, as well as documentation on tools and best practices. The main target audience are the Skyscrapers customers.

In this knowledge base you'll find information on how to use your Kubernetes cluster, how to create a Concourse pipeline, how to manage containerized applications and how to monitor them, etc. You'll also find some opinionated good practices, tips and recommendations, along with references to external sources of documentation and articles. It's intended to be browsed mainly through GitHub or a local git clone, so the file tree acts as the table of contents.

## Table of contents

Each header points to the main README of each platform offering. Within each folder you'll also find documentation on more advanced and/or specific topics.

- AWS
  - [IAM Delegated Access](aws/iam_delegated_access.md)
- [Backups](backups.md)
- [Coding Guidelines and Best Practices](coding_guidelines/README.md) - Main coding guidelines we adhere to at Skyscrapers
  - [Concourse](coding_guidelines/concourse.md)
  - [Docker](coding_guidelines/docker.md)
  - [Documents](coding_guidelines/documents.md)
  - [Git](coding_guidelines/git.md)
  - [Helm](coding_guidelines/helm.md)
  - [Teragrunt / OpenTofu](coding_guidelines/terragrunt.md)
- [Concourse](Concourse/README.md) - Walk-through on how to use Concourse and setup pipelines
  - [Feature Environments](Concourse/feature_environments.md)
- [Kubernetes](kubernetes/README.md) - Walk-through on getting access to and deploying applications on our K8s clusters
  - [Backups](kubernetes/backups.md)
  - [Cluster Balancing](kubernetes/cluster_balancing.md)
  - [Dynamic, whitelabel-style Ingress to your application](kubernetes/create_ingress_via_api.md)
  - [Helm (v3)](kubernetes/helm.md)
  - [Logs](kubernetes/logging.md)
  - [Monitoring](kubernetes/monitoring.md)
  - [Network Policies](kubernetes/network_policies.md)
  - [OpenVPN](kubernetes/openvpn.md)
  - [Persistent Volumes](kubernetes/persistent_volumes.md)
  - [Pod Autoscaling](kubernetes/pod_autoscaling.md)
  - [Pod Disruptions](kubernetes/pod_disruptions.md)
  - [Role Based Access Control (RBAC)](kubernetes/RBAC.md)
  - [Service Mesh](kubernetes/service_mesh.md)
  - [UDP](kubernetes/udp.md)
  - [Vault](kubernetes/vault.md)
- [MongoDB Atlas](mongodb_atlas/README.md)
- [Onboarding](onboarding/README.md) - All resources related to the onboarding track
  - [Kubernetes Platform Orientation](onboarding/orientation.md) - Kubernetes Platform Orientation workshop material
- [Runbook](runbook.md) - Description of our custom monitoring alerts with possible remediations

## Contributing

Your contributions are very welcomed, so if you have any suggestions to improve this documentation feel free to open an issue or a pull request. In any case, it's important that you keep in mind the following guidelines:

- All documentation in this repo is written in Markdown and must be compliant with [GitHub Flavoured Markdown (GFM)](https://guides.github.com/features/mastering-markdown/#GitHub-flavored-markdown)
- File and folder names should be descriptive enough to let the reader know what's in them
- Make sure to update the main ToC above
