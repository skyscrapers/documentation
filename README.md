# Skyscrapers products & services documentation

This repository contains user-level technical documentation for our product & service offering, documentation on tools and best practices as well as the way we do our work (processes). The main target audience are the Skyscrapers customers.

## Technical Documentation

In this knowledge base you'll find information on how to use your Kubernetes cluster, how to create a Concourse pipeline, how to manage containerized applications and how to monitor them, etc. You'll also find some opinionated good practices, tips and recommendations, along with references to external sources of documentation and articles. It's intended to be browsed mainly through GitHub or a local git clone, so the file tree acts as the table of contents.

Each header points to the main README of each platform offering. Within each folder you'll also find documentation on more advanced and/or specific topics.

- AWS
  - [ECR](aws/ecr.md)
  - [IAM Delegated Access](aws/iam_delegated_access.md)
  - [RDS](aws/RDS.md)
- [Backups](backups.md)
- [Coding Guidelines and Best Practices](coding_guidelines/README.md) - Main coding guidelines we adhere to at Skyscrapers
  - [Concourse](coding_guidelines/concourse.md)
  - [Docker](coding_guidelines/docker.md)
  - [Documents](coding_guidelines/documents.md)
  - [Git](coding_guidelines/git.md)
  - [Helm](coding_guidelines/helm.md)
  - [Terragrunt / OpenTofu](coding_guidelines/terragrunt.md)
- [Concourse](Concourse/README.md) - Walk-through on how to use Concourse and setup pipelines
  - [Feature Environments](Concourse/feature_environments.md)
- [Kubernetes](kubernetes/README.md) - Walk-through on getting access to and deploying applications on our K8s clusters
  - [Getting started](kubernetes/getting_started.md)
  - [Authentication](kubernetes/authentication.md)
  - [AWS Secrets Manager](kubernetes/aws_secrets_manager.md)
  - [Backups](kubernetes/backups.md)
  - [Dynamic, whitelabel-style Ingress to your application](kubernetes/create_ingress_via_api.md)
  - [Flux](kubernetes/flux.md)
  - [Dockerhub](kubernetes/dockerhub.md)
  - [GitHub actions runners](kubernetes/github-actions-runner-controller.md)
  - [Helm](kubernetes/helm.md)
  - [Jaeger](kubernetes/jaeger.md)
  - [Karpenter](kubernetes/karpenter.md)
  - [Logs](kubernetes/logging.md)
  - [Monitoring](kubernetes/monitoring.md)
  - [Network Policies](kubernetes/network_policies.md)
  - [OpenVPN](kubernetes/openvpn.md)
  - [Persistent Volumes](kubernetes/persistent_volumes.md)
  - [Pods](kubernetes/pods.md)
  - [Role Based Access Control (RBAC)](kubernetes/RBAC.md)
  - [Service Mesh](kubernetes/service_mesh.md)
  - [UDP](kubernetes/udp.md)
  - [Using GPU accelerated instances](kubernetes/using_gpu_accelerated_instances.md)
  - [Vault](kubernetes/vault.md)
- [MongoDB Atlas](mongodb_atlas/README.md)
- [Runbook](runbook.md) - Description of our custom monitoring alerts with possible remediations

## Onboarding

Here you will find the resources related to the onboarding track for our new customers. 

- [Onboarding](onboarding/README.md)
  - [Kubernetes Platform Orientation](onboarding/orientation.md) - Kubernetes Platform Orientation workshop material

## Process and services

This is where you can find information on certain aspects of the DOaaS that we offer (.e.g. the way we offer our every day support to you).

- [DevOps-as-a-Service: Platform Users and User Roles](Process_and_Services/platformusersandroles.md)
- [Skyscrapers Support Process](support_process.md)
- [Skyscrapers Security Policy](Process_and_Services/security_policy.md)

## Contributing

Your contributions are very welcomed, so if you have any suggestions to improve this documentation feel free to open an issue or a pull request. In any case, it's important that you keep in mind the following guidelines:

- All documentation in this repo is written in Markdown and must be compliant with [GitHub Flavoured Markdown (GFM)](https://guides.github.com/features/mastering-markdown/#GitHub-flavored-markdown)
- File and folder names should be descriptive enough to let the reader know what's in them
- Make sure to update the main ToC above
