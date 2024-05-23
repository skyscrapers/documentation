# Helm

- [Helm](#helm)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
    - [Install Helm on Windows](#install-helm-on-windows)
    - [Install Helm on macOS](#install-helm-on-macos)
    - [Install Helm on Linux](#install-helm-on-linux)
  - [Using Helm](#using-helm)
    - [Adding a Repository](#adding-a-repository)
    - [Searching for Charts](#searching-for-charts)
    - [Installing a Chart](#installing-a-chart)
    - [Upgrading a Release](#upgrading-a-release)
    - [Rolling back a Release](#rolling-back-a-release)
    - [Uninstalling a Release](#uninstalling-a-release)
    - [Listing Releases](#listing-releases)
  - [Interacting with Helm](#interacting-with-helm)
    - [Helm Commands](#helm-commands)
    - [Helm Configuration](#helm-configuration)
    - [Working with Helm Hooks](#working-with-helm-hooks)
  - [Troubleshooting](#troubleshooting)
    - [Common Issues](#common-issues)
    - [Debugging Tips](#debugging-tips)
  - [Conclusion](#conclusion)

## Overview

Helm is a package manager for Kubernetes, providing a simple and efficient way to manage Kubernetes applications. It allows users to define, install, and upgrade even the most complex Kubernetes applications. Helm uses a packaging format called charts, which are collections of files that describe a related set of Kubernetes resources.

## Prerequisites

Before installing Helm, ensure you have the following:

- Kubernetes cluster (v1.16+)
- `kubectl` installed and configured
- Helm client (v3+)

## Installation

### Install Helm on Windows

1. **Download Helm:**

   ```sh
   choco install kubernetes-helm
   ```

2. **Verify the Installation:**

   ```sh
   helm version
   ```

### Install Helm on macOS

1. **Download Helm using Homebrew:**

   ```sh
   brew install helm
   ```

2. **Verify the Installation:**

   ```sh
   helm version
   ```

### Install Helm on Linux

1. **Download the Helm binary through the distro's package manager**

   ```sh
   curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
   sudo apt-get install apt-transport-https --yes
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
   sudo apt-get update
   sudo apt-get install helm
   ```

> [!NOTE]
> Alternatively download it from GitHub <https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3>

1. **Verify the Installation:**

   ```sh
   helm version
   ```

## Using Helm

### Adding a Repository

1. **Add the Stable Helm Repository:**

   ```sh
   helm repo add stable https://charts.helm.sh/stable
   ```

2. **Update Your Repository:**

   ```sh
   helm repo update
   ```

### Searching for Charts

1. **Search for a Chart:**

   ```sh
   helm search repo <chart-name>
   ```

### Installing a Chart

1. **Install a Chart:**

   ```sh
   helm install <release-name> <chart-name>
   ```

   Example:

   ```sh
   helm install my-release stable/mysql
   ```

### Upgrading a Release

1. **Upgrade a Release:**

   ```sh
   helm upgrade <release-name> <chart-name>
   ```

   Example:

   ```sh
   helm upgrade my-release stable/mysql
   ```

### Rolling back a Release

1. **Rolling back a Release:**

   ```sh
   helm rollback <release-name> <revision>
   ```

   Example:

   ```sh
   helm rollback my-release 2
   ```

### Uninstalling a Release

1. **Uninstall a Release:**

   ```sh
   helm uninstall <release-name>
   ```

   Example:

   ```sh
   helm uninstall my-release
   ```

### Listing Releases

1. **List All Releases:**

   ```sh
   helm list
   ```

## Interacting with Helm

### Helm Commands

- **Install a Chart:**

  ```sh
  helm install <release-name> <chart-name> --values <values.yaml>
  ```

- **Get Release Information:**

  ```sh
  helm status <release-name>
  ```

- **Rollback a Release:**

  ```sh
  helm rollback <release-name> <revision>
  ```

- **Show the Values File for a Chart:**

  ```sh
  helm show values <chart-name>
  ```

### Helm Configuration

- **Override Default Values:**
  You can create a `values.yaml` file to override default values:

  ```yaml
  replicaCount: 2
  image:
    repository: myrepo/myimage
    tag: latest
  ```

- **Install with Custom Values:**

  ```sh
  helm install <release-name> <chart-name> -f <path-to-values-file>
  ```

### Working with Helm Hooks

Helm hooks allow you to intervene at certain points in a release lifecycle, such as before installing or after upgrading. For more details, refer to the [Helm Hooks documentation](https://helm.sh/docs/topics/charts_hooks/).

## Troubleshooting

### Common Issues

- **Failed to install chart:**
  Ensure the chart name and repository are correct and updated:

  ```sh
  helm repo update
  helm search repo <chart-name>
  ```

- **Release is in a failed state:**
  Check the release status and logs for more information:

  ```sh
  helm status <release-name>
  kubectl logs <pod-name>
  ```

### Debugging Tips

- **Verbose Output:**
  Use the `--debug` flag with Helm commands to get detailed output:

  ```sh
  helm install <release-name> <chart-name> --debug
  ```

## Conclusion

Helm simplifies managing Kubernetes applications by providing powerful commands and an easy-to-use interface. By following this guide, you should be able to install, use, and interact with Helm efficiently. For more information and advanced usage, refer to the [official Helm documentation](https://helm.sh/docs/).
