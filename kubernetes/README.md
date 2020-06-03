<!-- markdownlint-disable no-inline-html -->

# Walk-through of the Skyscrapers' Kubernetes cluster

Here you'll find user-level documentation of our Kubernetes reference solution. You'll learn how to access the cluster, how to deploy applications and services and how to monitor them. You'll also find some tips & tricks on how to get the most from your Kubernetes cluster.

The best place to start is this walk-through. After that you can read more in-depth documentation on specific topics in their respective files in this folder.

- [Walk-through of the Skyscrapers' Kubernetes cluster](#walk-through-of-the-skyscrapers-kubernetes-cluster)
  - [Requirements](#requirements)
  - [Authentication](#authentication)
  - [Deploying applications & services on Kubernetes: the Helm Package Manager](#deploying-applications--services-on-kubernetes-the-helm-package-manager)
  - [Ingress](#ingress)
    - [HTTP traffic (ports 80 and 443)](#http-traffic-ports-80-and-443)
    - [Other traffic](#other-traffic)
    - [Dynamic, whitelabel-style Ingress to your application](#dynamic-whitelabel-style-ingress-to-your-application)
  - [DNS](#dns)
  - [Automatic SSL certificates](#automatic-ssl-certificates)
    - [Examples](#examples)
      - [Get a LetsEncrypt certificate using defaults (dns01)](#get-a-letsencrypt-certificate-using-defaults-dns01)
      - [Get a LetsEncrypt certificate using the http01 challenge](#get-a-letsencrypt-certificate-using-the-http01-challenge)
      - [Get a LetsEncrypt wildcard certificate the dns01 challenge](#get-a-letsencrypt-wildcard-certificate-the-dns01-challenge)
  - [IAM Roles](#iam-roles)
  - [Persistent Volumes](#persistent-volumes)
  - [Monitoring](#monitoring)
  - [Cluster updates & rollouts](#cluster-updates--rollouts)
  - [Cronjobs](#cronjobs)
    - [Cronjob Monitoring](#cronjob-monitoring)
    - [Clean up](#clean-up)

## Requirements

- [kubectl](https://kubernetes.io/docs/tasks/kubectl/install/)
- [helm](https://docs.helm.sh/using_helm/#installing-helm)
- [AWS cli >= 1.16.156](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

**Note**: You can also easily install all the above via [Homebrew](https://brew.sh/) or [Linuxbrew](https://docs.brew.sh/Linuxbrew) on Linux, macOS and Windows (through [WSL](https://docs.microsoft.com/en-us/windows/wsl/about)): `brew install awscli kubernetes-cli kubernetes-helm`

## Authentication

To gain access to an EKS cluster you need to authenticate via AWS IAM and configure your kubeconfig accordingly. To do this you'll need a recent version of `awscli` (`>= 1.16.156`). If you don't have the AWS CLI yet, you can install it by [following the AWS instructions](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) or via [Homebrew/Linuxbrew](https://brew.sh/):

```bash
brew install awscli
```

You'll first need to authenticate to the AWS account where the EKS cluster is deployed (or your Admin account if you use delegation). Depending on how you configured your `awscli` config, `--region` and `--profile` are optional.

Make sure to replace `<my_assumed_role_arn>` with a correct role depending on your access level. Which roles you can assume are documented in your customer-specific documentation.

```bash
aws eks update-kubeconfig --name <cluster_name> --alias <my_alias> [--role-arn <my_assumed_role_arn>] [--region <aws_region>] [--profile <my_aws_profile>]

# For example:
aws eks update-kubeconfig --name production-eks-example-com --alias production --role-arn arn:aws:iam::123456789012:role/developer
```

To access the Kubernetes dashboard, you'll need to [install `kauthproxy`](https://github.com/int128/kauthproxy). You can install this through brew on macOS or [via `Krew`](https://github.com/kubernetes-sigs/krew) on any OS.

Example via Krew:

```bash
# Install Krew via brew
brew install krew

# Make sure to add the krew bin folder to your path, for example (bash):
# echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Install kauthproxy via Krew
kubectl krew install auth-proxy

# Access the Kubernetes dashboard. This should open your browser window automatically
kubectl auth-proxy -n kubernetes-dashboard https://kubernetes-dashboard.svc

# To make things easier to remember, you could add an `alias` to your shell's config
alias kube-dashboard='kubectl auth-proxy -n kubernetes-dashboard https://kubernetes-dashboard.svc'
```

## Deploying applications & services on Kubernetes: the Helm Package Manager

After a roll out of a Kubernetes cluster, it could be tempting to start executing numerous `kubectl create` or `kubectl apply` commands to get stuff deployed on the cluster.

Running such commands is a good idea to learn how deployments are done on Kubernetes, but it **is not the appropriate tool** to construct fully self contained application deployments. The Kubernetes community came up with a separate tool for that:

[The Helm Package Manager](https://helm.sh/)

With Helm, you create self contained packages for a specific piece of a deployment, e.g. a web stack. Such packages are called `Charts` in Helm terminology.

You probably want such a stack to be deployed in a specific Kubernetes namespace, with a specific configuration (`ConfigMap`), defining a Kubernetes `Service` referring to a `Deployment`. But if you want to have this setup reproducible, you need a way to parameterize this.
By using a Template engine, a [Go function library](http://masterminds.github.io/sprig/) and the use of a `Values.yaml` file, you can build a template of a specific piece and re-use that for multiple deployments.

**Important**: It's worth noting that [Tiller won't be deployed anymore on K8s clusters](https://changelog.skyscrapers.eu/kubernetes/2020/04/10/helm3.html), which means that you should be using Helm v3. Please check out [our dedicated Helm page on how to migrate your releases from Helm v2 to v3](helm.md).

The [Helm documentation](https://helm.sh/docs/) is quite good and explanatory, and the [best practices section](https://helm.sh/docs/chart_best_practices/) highlight some of the important topics around chart development.

Good examples always help out a lot. Here is a list of existing git Charts repositories:

- [Kubernetes Charts](https://github.com/helm/charts/)
- [Skyscrapers Charts](https://github.com/skyscrapers/charts)

The above Chart repositories contain Charts that serve as building blocks for bigger composite installations.

## Ingress

> [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) is deployed by default.

By default we deploy an Ingress controller which exposes services to the public Internet. We also provide the option to deploy an internal-only controller for exposing your K8s services within the private AWS VPC.

### HTTP traffic (ports 80 and 443)

At the moment, HTTP (ports 80 and 443) ingress to the cluster is done as follows:

```console
Public ELB -> NGINX Ingress -> Pod Endpoints (through Service selectors)
```

To make your deployment accessible from the outside world through HTTP(S), you need to create an `Ingress` object with the following annotation: `kubernetes.io/ingress.class: "nginx"`. This will tell the Nginx ingress controller to route traffic to your services. You can find more information on how the Nginx ingress controller works and some examples in the [official documentation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/).

To use the internal-only Ingress, you need to set the `kubernetes.io/ingress.class: "nginx-internal"` annotation on your `Ingress`.

### Other traffic

If your application needs to be accessible through other ports or through TCP, you'll need to create your own ingresses. Normally you'll want to create a `Service` of type `LoadBalancer`. With that, Kubernetes will automatically create an ELB that will route traffic to your pods on the needed ports. An example of this kind of `Service` can be found [here](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer).

*Please note that launching extra ELBs increases your AWS costs.*

### Dynamic, whitelabel-style Ingress to your application

If your application allows for end-customers to use their custom domain, you can let your application interface directly with the K8s API to manage `Ingress` objects. For more info, check our [separate page on the subject](/kubernetes/create_ingress_via_api.md).

## DNS

> [ExternalDNS](https://github.com/kubernetes-incubator/external-dns) is deployed by default.

It will automatically configure DNS records in Route 53 for your application. For your Ingress, it well add records for the `host` field by default.

To add a record to a `Service` object, you can use the following annotation:

```yaml
external-dns.alpha.kubernetes.io/hostname: nginx.example.org.
```

The only requirement for this to work is that the DNS zone for the domain is hosted in Route53 in the same AWS account as the cluster.

## Automatic SSL certificates

> [Cert-manager](https://cert-manager.io/docs/) is deployed by default.

As with the DNS records, SSL certificates can also be automatically fetched and setup for applications deployed on the Kubernetes cluster via [`cert-manager`](https://github.com/jetstack/cert-manager/). We deploy a `letsencrypt-prod` [`ClusterIssuer`](https://cert-manager.io/docs/concepts/issuer/) by default, which uses `dns01` validation via Route 53 (used in conjunction with ExternalDNS).

To use this `ClusterIssuer` and get a certificate for your application, you simply need to add the following annotation to the `Ingress` object:

```yaml
kubernetes.io/tls-acme: "true"
```

You'll also need to add a `tls` section in the spec of the `Ingress` object, like the following:

```yaml
tls:
  - secretName: example-application-com-tls
    hosts:
      - example.application.com
```

With the `hosts` array of the `tls` section you're telling the cluster which domains need to be in the certificate, and in which `Secret` it should store the private key.

Of course you can also define your own [`Issuers`](https://cert-manager.readthedocs.io/en/latest/reference/issuers.html) and/or [`ClusterIssuers`](https://cert-manager.readthedocs.io/en/latest/reference/clusterissuers.html).

You can get a list of all issued certificates:

```sh
$ kubectl get certificates --all-namespaces
NAMESPACE        NAME                                 CREATED AT
infrastructure   cert-manager-webhook-ca              9m
infrastructure   cert-manager-webhook-webhook-tls     9m
infrastructure   foo-staging-cert                     2m
infrastructure   kubesignin-alertmanager-lego-tls     5m
infrastructure   kubesignin-dashboard-lego-tls        5m
infrastructure   kubesignin-grafana-lego-tls          5m
infrastructure   kubesignin-kibana-lego-tls           5m
infrastructure   kubesignin-kubesignin-app-lego-tls   5m
infrastructure   kubesignin-prometheus-lego-tls       5m
infrastructure   wild-staging-cert                    37s
```

### Examples

Below are some simple examples on how to issue certicicates as usually done on the `Ingress`.

There are way more possibilities than described in the examples, which you can find in the [official documentation](https://cert-manager.io/docs/).

#### Get a LetsEncrypt certificate using defaults (dns01)

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
    kubernetes.io/tls-acme: "true"
  name: foo
  namespace: default
spec:
  rules:
    - host: foo.staging.skyscrape.rs
      http:
        paths:
          - backend:
              serviceName: foo
              servicePort: http
            path: /
  tls:
    - secretName: foo-staging-tls
      hosts:
        - foo.staging.skyscrape.rs
```

#### Get a LetsEncrypt certificate using the http01 challenge

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
    kubernetes.io/tls-acme: "true"
  labels:
    use-http-solver: "true"
  name: bar
  namespace: default
spec:
  rules:
    - host: bar.staging.skyscrape.rs
      http:
        paths:
          - backend:
              serviceName: bar
              servicePort: http
            path: /
  tls:
    - secretName: bar-staging-tls
      hosts:
        - bar.staging.skyscrape.rs
```

#### Get a LetsEncrypt wildcard certificate the dns01 challenge

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
    kubernetes.io/tls-acme: "true"
  name: lorem
  namespace: default
spec:
  rules:
    - host: lorem.staging.skyscrape.rs
      http:
        paths:
          - backend:
              serviceName: lorem
              servicePort: http
            path: /
  tls:
    - secretName: wildcard-staging-skyscrape-rs-tls
      hosts:
        - '*.staging.skyscrape.rs'
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
    kubernetes.io/tls-acme: "true"
  name: ipsum
  namespace: default
spec:
  rules:
    - host: ipsum.staging.skyscrape.rs
      http:
        paths:
          - backend:
              serviceName: ipsum
              servicePort: http
            path: /
  tls:
    - secretName: wildcard-staging-skyscrape-rs-tls
      hosts:
        - '*.staging.skyscrape.rs'
```

You could also issue a `Certificate` first to re-use that later in your `Ingresses`:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: wildcard-staging-skyscrape-rs
  namespace: default
spec:
  secretName: wildcard-staging-skyscrape-rs-tls
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-prod
  dnsNames:
    - 'skyscrape.rs'
    - '*.skyscrape.rs'
```

**Note**: While it is possible to generate multiple wildcard certificates via a different `secretName`, it is advised / more efficient to reuse the same `Secret` for all ingresses using the wildcard.

**Note 2**: A `Secret` is scoped within a single `Namespace`, which means if you want to use a wildcard certificate in another `Namespace` cert-manager will request and validate a new certificate from LetsEncrypt. (unless you replicate the `Secrets`)

## IAM Roles

> [Kube2iam](https://github.com/jtblin/kube2iam) is deployed by default.

Your deployments can be assigned with specific IAM roles to grant them fine-grained permissions to AWS services. To do that just add the following annotation to `.spec.template.metadata.annotations` in your `Deployment` objects:

```yaml
iam.amazonaws.com/role: arn:aws:iam::123456789:role/kube2iam/something
```

You can find some examples in the `kube2iam` official documentation: <https://github.com/jtblin/kube2iam#kubernetes-annotation>

**Note**: when you create a new IAM role for a Kubernetes deployment, make sure you create it under the `/kube2iam/` path, otherwise your deployments won't be able to assume it. Also you'll need to add a [trust relationship](http://docs.aws.amazon.com/directoryservice/latest/admin-guide/edit_trust.html) to your role so the k8s nodes roles can assume it. Normally the k8s nodes role will have the following name scheme: `arn:aws:iam::<aws-account-id>:role/nodes.<cluster-domain-name>`

## Persistent Volumes

[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) in our cluster are backed by AWS EBS volumes. Among the obvious caveats around scheduling (a volume is limited to a single AZ), there's also a more silent and hard to predict caveat.

[Depending on EC2 instance type, most of them support a maximum of only 28 attachments, including network interfaces, EBS volumes, and NVMe instance store volumes.](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/volume_limits.html#instance-type-volume-limits). This means that only a limited number of EBS volumes per K8s node can be used, also considering our CNI uses multiple network interfaces.

Kubernetes [limits the max amount of volumes for M5,C5,R5,T3 and Z1D to only 25 volumes to be attached to a Node](https://kubernetes.io/docs/concepts/storage/storage-limits/#dynamic-volume-limits), however this often isn't enough depending how much network interfaces are in use by the CNI.

Unfortunately AWS doesn't throw an error either when this happens. Instead a Volume will stay stuck in the `Attaching` state and your Pod will fail to launch. After ~5 minutes Kubernetes will [taint the node](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) with `NodeWithImpairedVolumes`.

We have added a Prometheus alert to catch this taint and you can [follow the actions described in the runbook](https://github.com/skyscrapers/documentation/tree/master/runbook.md#alert-name-nodewithimpairedvolumes) when this happens.

## Monitoring

Cluster and application monitoring is a quite extensive topic by itself, so there's a specific document for it [here](./monitoring.md).

## Cluster updates & rollouts

As part of our responsibilities, we continuously roll improvements (upgrades, updates, bug fixes and new features). Depending on the type of improvement, the impact on platform usage and application varies anywhere between nothing to having (a small) downtime. Below an overview for the most common types of improvements. More exceptional types will be handled separately.

| Type of improvement | Description | Expected impact on your workloads |
|:-------------------:|:-----------|:---------------------------------:|
| Add-ons (non-breaking) | Improvements expected to not have an impact on the current usage of the cluster or application behaviour.<br/><br/>These are rolled out automatically at any time during the day. You are informed during the updates. | No impact |
| Add-ons (non-breaking but disruptive) | Improvements to add-ons that may lead to temporary unavailability of platform functionalities (monitoring, logging, dashboard, etc) but that do not impact application workloads.<br/><br/>These are rolled out automatically at any time during the day. You are informed before and during the updates. | No impact |
| Add-ons (breaking) | These improvements may need changes or intervention by you before they can be rolled out.<br/><br/>We will reach out to you to discuss what’s needed on how the improvement will be rolled out. | In some cases: minimal planned downtime |
| Cluster improvements | Low-frequency improvements to the foundations of the cluster. Usually these involve rolling updates leading to nodes being recycled.<br/><br/>These are rolled out automatically at any time during the day. You are informed before and during the updates. | Cluster-aware workloads: No impact<br/><br/>Other workloads: potential minimal downtime |

To minimize the impact on your workloads, we suggest you to implement cluster-aware workloads as much as possible (TODO: define cluster-aware) and implement `PodDisruptionBudgets`. There's more information on this [here](./pod_disruptions.md).

## Cronjobs

Kubernetes can run cronjobs for you. More information/examples about cronjobs can be found [here](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).

### Cronjob Monitoring

Monitoring for cronjobs is implemented by default. This is done with prometheus and will alert when the last run of the cronjob has failed.

The following alerts are covering different failure cases accordigly:

- `KubeJobCompletion`: Warnning alert after 1 hour if any *Job* doesn't succeed or doesn't run at all.
- `KubeJobFailed`: Warning alert after 1 hour if any *Job* failed
- `KubeCronJobRunning`: Warning alert after 1 hour if a *CronJob* keeps on running

### Clean up

Starting from Kubernetes 1.7 the scheduled jobs don't get automatically cleaned up. So make sure that you add the following two lines to the `spec` section of your cronjob.

```yaml
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 3
```

This will clean up all jobs except the last 3, both for successful and failed jobs.
