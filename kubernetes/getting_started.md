# Getting Started

If you are new to Kubernetes, here you can find a guide to common Kubernetes concepts that you will need to deploy your first application.

## Command-line Tool

First you need to setup the kubernetes CLI tool called `kubectl` as stated in [the main requirements section](README.md#requirements), check [kubectl overview](https://kubernetes.io/docs/reference/kubectl/overview/
) for more information (do not necessarily go through all the details in here, but just a general overview to get yourself familiar with kubectl).

## General Concepts

Here you can find an introduction to the main components needed to deploy and run your first Kubernetes workload: <https://kubernetes.io/docs/concepts/workloads/>

## Networking

While the above step explains how a container should be deployed, it does not yet allow accessing it from the internet, here is how the container can be exposed and selected for receiving requests:

- <https://kubernetes.io/docs/concepts/services-networking/service/>

- <https://kubernetes.io/docs/concepts/services-networking/ingress/>

## More Configurations

The following are some important configuration concepts that your application will need to run:

- you can add environment variables to access inside the application: <https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/>

- configure your sensitive credentials with secrets: <https://kubernetes.io/docs/concepts/configuration/secret/>  

- manage how much resources your application can use: <[https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)>

## Package Management

Now that you know about all of these kubernetes resources, it's time to tie everything together with Helm, the package manager.

- You can find why we recommend using Helm to deploy your application [here](README.md#deploying-applications--services-on-kubernetes-the-helm-package-manager)

- Tutorial for creating a helm chart: <https://helm.sh/docs/chart_template_guide/getting_started/>

- Introduction on Helm Values and how to use them: <https://helm.sh/docs/chart_template_guide/values_files/>
