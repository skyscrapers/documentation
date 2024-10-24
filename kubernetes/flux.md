# Flux

> [!NOTE]
> Primary audience: Skyscrapers customers, Skyscrapers internal

## Introduction

[Flux](https://fluxcd.io/) is a way to deploy and maintain your applications and components through [GitOps](https://www.gitops.tech/#what-is-gitops). It is designed to keep your Kubernetes clusters in sync based on the configuration in git and to automate updates to configuration when Flux detects it. This documentation provides guidance on setting up and managing your repository structure using Flux in the cooperation with Skyscrapers.

In short, this means Flux will pull your changes from Git and keep everything reconciled. For example if you commit a neww container image to your helm chart, Flux will detect this and perform a helm upgrade directly from within the Kubernetes cluster. You are also not limited to Helm, Flux works great with Kustomize too.

- [Flux](#flux)
  - [Introduction](#introduction)
  - [Initial setup](#initial-setup)
  - [Repository Structure](#repository-structure)
    - [Directory Breakdown](#directory-breakdown)
    - [Applications](#applications)
  - [Example Configuration](#example-configuration)
    - [kustomization.yaml](#kustomizationyaml)
    - [apps.yaml](#appsyaml)
      - [Internal within the same repository](#internal-within-the-same-repository)
      - [External repository](#external-repository)
  - [Monitoring](#monitoring)
    - [Slack Alerts](#slack-alerts)
    - [Grafana Dashboards](#grafana-dashboards)
      - [Flux Cluster Stats](#flux-cluster-stats)
      - [Flux Control Plane](#flux-control-plane)
  - [ECR access](#ecr-access)

## Initial setup

If you think this could be useful for your team, get in touch with us so we can enable it on your cluster(s), and offer you guidance and training on how to leverage it for your use-case.

Once the initial bootstrap is done, you can start managing your applications and infrastructure using Flux based.

## Repository Structure

The recommended structure for organizing your Flux configurations and related resources in the repository is as follows:

```bash
docs/
flux/
├── apps/
    ├── base/
    ├── production/
    └── staging/
└── clusters/
    ├── production-cluster-name/
        ├── flux-system/
            └── ...
        └── apps.yaml
    └── staging-cluster-name/
        ├── flux-system/
            └── ...
        └── apps.yaml
k8s-clusters/
├── production-cluster-name.yaml
└── staging-cluster-name.yaml
terraform/
├── live/
└── modules/
```

### Directory Breakdown

**flux/**: Directory to hold all Flux-related configuration files.

- **apps/**: Optional directory to organize application configurations.
  - **base/**: Base configurations for all environments.
  - **production/**: Specific configurations for the production environment.
  - **staging/**: Specific configurations for the staging environment.
- **clusters/**: Directory for managing cluster-specific configurations.
  - **production-cluster-name/**: Configuration for the production cluster.
    - **flux-system/**: Contains the core Flux system configurations. This is automatically provisioned for you by Skyscrapers/Flux and can't be modified manually.
    - **apps.yaml**: Kustomization overlay pointing to the apps folder or repository. See [Applications section](#applications) for more information.
  - **staging-cluster-name/**: Configuration for the staging cluster, structured similarly to the production cluster.

### Applications

To allow for more flexibility for your applications, you can point to a separate repository in the `apps.yaml`. This way you can still manage your application manifests within your own git repository, while still having Flux deploy them. The only requirement for this is that you define a `GitRepository` and create a Secret in the `flux-system` namespace with an [SSH deploy key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys) to access the repository. Example:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-apps
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: apps-deploykey
  url: ssh://git@github.com/<organisation>/<repo-name>.git
```

## Example Configuration

This section provides some examples of the configuration files used to manage Flux resources. A for more in depth explanation of the configuration files, please refer to the [Flux documentation](https://fluxcd.io/docs/) or reach out to us.

### kustomization.yaml

The `kustomization.yaml` file is used to manage resources in a structured way. In this file you refer to all files that you want Kustomize to include. An example configuration might look like:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - monitoring.yaml
  - scaling.yaml
```

### apps.yaml

This file points to the applications folder or repository, allowing Flux to manage application deployments. An example configuration might be:

#### Internal within the same repository

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./flux/apps/production
  sourceRef:
    kind: GitRepository
    name: flux-system
```

#### External repository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app
spec:
  url: ssh://git@github.com/<organisation>/<repo-name>.git
  ref:
    branch: main
  secretRef:
    name: apps-deploykey
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: app
spec:
  sourceRef:
    kind: GitRepository
    name: app
  path: flux/app/production
```

> [!IMPORTANT]
> Requirement for this is that a secret (called `apps-deploykey` here in this example) is deployed in the `flux-system` namespace with an [SSH deploy key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys) to access the repository.

## Monitoring

### Slack Alerts

Out of the box Flux alerts in the shared Slack channel for errors.
The destination channel and severity can be configured to your liking. Please reach out to us if you want to change the configuration or add more alerts.

![Slack alert of Flux](./images/flux_slack.png)

### Grafana Dashboards

There are 2 Grafana dashboards available in your Grafana for monitoring Flux:

#### Flux Cluster Stats

This dashboard provides an overview on the overall status of all objects managed by Flux.

![Flux Cluster Stats](./images/flux_cluster_stats.png)

#### Flux Control Plane

This dashboard provides an overview of the Flux system components and their health status.

![Flux Control Plane](./images/flux_control_plane.png)

## ECR access

> [!NOTE]
> Primary audience: Skyscrapers internal

If AWS ECR is used as registry, Skyscrapers commonly needs to setup cross-account access to ECR repositories. Instructions can be found in the [AWS ECR documentation](/aws/ecr.md#setup-cross-account-ecr-access).
