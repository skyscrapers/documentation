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
  - [Writing Helm charts](#writing-helm-charts)
    - [Chart.yaml file](#chartyaml-file)
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

## Writing Helm charts

Writing Helm charts involves creating a directory structure with the necessary files to define the Kubernetes resources. For more information on how the directory structure should look like, refer to the [Helm Chart documentation](https://helm.sh/docs/topics/charts/).

We recommend initializing a new charts using the `helm create` command:

```sh
helm create my-awesome-chart
```

This command creates a new directory called `my-awesome-chart` with the following structure:

```plaintext
my-awesome-chart/
├── Chart.yaml
├── charts/
├── templates/
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   └── <resources>.yaml
├── values.yaml
├── .helmignore
├── .gitignore
├── README.md
└── LICENSE
```

The best practice for writing Helm charts is to follow the [Helm Best Practices](https://helm.sh/docs/chart_best_practices/). Kubernetes resources should be in separate  `.yaml` files, but logical bundling of resources is welcome: Eg: All services in one file, all deployments in another file ,etc. Resources must be split by the `---` delimiter.
.


## Handling CustomResourceDefinitions (CRDs)

For handling CustomResourceDefinitions (CRDs), there are two common approaches.

**We recommend the first approach**:

The first one is using [a separate chart to manage them](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/#method-2-separate-charts) from the Helm Best Practices documentation. This approach has more overhead (installing 2 charts instead of 1), but gives more clarity and control over the CRDs installation and upgrade cycle.

The second one is to package the CRDs with the main chart. This approach is simpler and more straightforward, but it can lead to issues when upgrading the CRDs. For more information, refer to the [Let Helm do it for you section (caveats explained)](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/#method-1-let-helm-do-it-for-you).


### Chart.yaml file

The `Chart.yaml` file contains metadata about the chart, such as the name, version, and description. A full example can be found [here](https://helm.sh/docs/topics/charts/#the-chartyaml-file).

Many fields of the `Chart.yaml` file are optional, but our recommendation is to fill in at least the following optional fields:

- `kubeVersion`: The Kubernetes version the chart is compatible with. Having this field helps users understand which Kubernetes versions the chart supports.
- `description`: A short description of the chart. This field helps users understand what the chart does.
- `type`: See [Chart Types](https://helm.sh/docs/topics/charts/#chart-types) for more information.
- `home` and `sources`: URLs to the project home and source code, respectively. These fields help users find more information about the chart.
- `dependencies`: A list of charts this chart depends on. This field helps users understand what other charts are required to use this chart.
- `appVersion`: The version of the app that this chart installs. This field tells us which version of the application the chart installs by default, if not specified in the `values.yaml` file.


### Naming of Resources

Our recommendation for naming of resources in Helm charts is the following:

- `metadata.name`: The name of the resource should be the same as the release name. This ensures that the resource name is unique and easy to identify.

In case of multiple of the same resource, a postfix should be added to the resource name. Eg: if you have multiple services, the name should be `{{ .Release.Name }}-{{ .Values.serviceName }}`. Eg: `my-awesome-chart-api`, `my-awesome-chart-frontend`.


### Additional Labels & Annotations

Adding labels should be done through the `values.yaml` file. This allows users to add more labels as needed. For example:

```yaml
additionalLabels:
  company: my-awesome-company
  additionalLabel: additionalValue
```

Including these labels in the resources is done by using the `{{ .Values.labels }}` template. For example:

```yaml
metadata:
  labels:
    {{- include "my-awesome-chart.labels" . | nindent 4 }} # Include labels from helpers.tpl
    {{- toYaml .Values.additionalLabels | nindent 4 }} # Include additional labels from values.yaml
```

The same approach can be used for annotations.

### Giving Options

Helm charts should be written in an environment agnostic way (To allow deployment on different cloud providers, on-prem, etc.). It is recommended to provide options in the `values.yaml` file and using [Flow Control](https://helm.sh/docs/chart_template_guide/control_structures/) in the resources themselves. For example, exposing your application through a LoadBalancer service in a cloud provider, but using a NodePort service in an on-prem environment.

```yaml
service:
  type: LoadBalancer # NodePort, LoadBalancer...
  port: 80
  targetPort: 80
  nodePort: null
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-awesome-chart.fullname" . }}
  ...
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
{{- if and (eq .Values.service.type "NodePort") (ne .Values.service.nodePort null) }}
      nodePort: {{ .Values.service.nodePort }}
{{- end }}
      protocol: TCP
  ...
```

### Using helpers.tpl

`helpers.tpl` is a file that contains [defined templates](https://helm.sh/docs/chart_best_practices/templates/#names-of-defined-templates) that are globally accessible in your chart. Our general recommendation is to not put too much logic in the `helpers.tpl` file, as it can make the chart harder to understand. Instead, the already generated templates in the `helpers.tpl` file should be used as much as possible.


### Using NOTES.txt

We reccomend using the `NOTES.txt` file to provide information to the user after the chart is installed. This file should contain information about how to access the application, how to configure it, etc. A good example is auto-generated when using `helm create`.

`NOTES.txt` file is optional, but it is a good practice to include it in your chart.



## Conclusion

Helm simplifies managing Kubernetes applications by providing powerful commands and an easy-to-use interface. By following this guide, you should be able to install, use, create Helm charts and interact with Helm efficiently. For more information and advanced usage, refer to the [official Helm documentation](https://helm.sh/docs/).
