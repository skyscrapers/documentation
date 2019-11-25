# Walk-through of the Skyscrapers' Kubernetes cluster

## Requirements

* [kubectl](https://kubernetes.io/docs/tasks/kubectl/install/)
* [helm](https://docs.helm.sh/using_helm/#installing-helm)
* [AWS cli >= 1.16.156](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

**Note**: You can also easily install all the above via [Homebrew](https://brew.sh/) or [Linuxbrew](https://docs.brew.sh/Linuxbrew) on Linux, macOS and Windows (through [WSL](https://docs.microsoft.com/en-us/windows/wsl/about)): `brew install awscli kubernetes-cli kubernetes-helm`

## Authentication (KOPS cluster)

### First time login

Once the cluster is setup:

1. Head to the url `https://kubesignin.<cluster-domain-name>/login`.
2. Select the appropriate login for your company (GitHub, GitLab, Google).
3. Authorize the Kubesignin / DEX application.
4. At this point you should have been redirected to a webpage containing a token (and some metadata). This token will be valid for one hour and will allow you to authenticate to the cluster. Ensure that there's an `email` claim in the metadata, otherwise you'll need to set a **public email address in your profile** at the Authentication backend (eg. Github).
5. Run `kubectl config set-credentials <username> --token=<token>` to set the token in kubectl config.
6. Run `kubectl config set-cluster <cluster-name> --server https://api.<cluster-domain-name> --certificate-authority=<certificate_file>` to set the cluster info in kubectl config. You can find the certificate data in the documentation in your repo.
7. Run `kubectl config set-context <anything-you-want> --cluster <cluster-name> --user <username>` to set the context in kubectl config.
8. Run `kubectl config use-context <anything-you-want>` to select the context created above. Contexts are useful when you want to manage multiple clusters.

Now you should be able to interact with the cluster. Try `kubectl get pods` for example (*Note that this command might return successfully but not display anything. This means that you don't have Pods deployed in the `default` namespace*)

### Subsequent logins

The tokens provided by kubesignin are only valid for 1 hour by default. This means that if you get a permission denied error while interacting with Kubernetes (eg./ through `kubectl`), you will have to refresh your token again by:

1. heading to the url `https://kubesignin.<cluster-domain-name>/login`.
2. running `kubectl config set-credentials <username> --token=<token>` to set the token in kubectl config.

## Authentication (EKS cluster)

To gain access to an EKS cluster you need to authenticate via AWS IAM and configure your kubeconfig accordingly. To do this you'll need a recent version of `awscli` (`>= 1.16.156`). If you don't have the AWS CLI yet, you can install it by [following the AWS instructions](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) or via [Homebrew/Linuxbrew](https://brew.sh/):

```bash
brew install awscli
```

You'll first need to authenticate to the AWS account where the EKS cluster is deployed. Depending on how you configured your `awscli` config, `--region` and `--profile` are optional.

Make sure to replace `<my_assumed_role_arn>` with a correct role depending on your access level. Which roles you can assume are documented in the customer-specific documentation.

```bash
aws eks update-kubeconfig --name <cluster_name> --alias <my_alias> --role-arn <my_assumed_role_arn> [--region <aws_region>] [--profile <my_aws_profile>]

# For example:
aws eks update-kubeconfig --name production-eks-example-com --alias production --role-arn arn:aws:iam::123456789012:role/developer
```

To access the Kubernetes dashboard, you'll need to [install `kauthproxy`](https://github.com/int128/kauthproxy). You can install this through brew on macOS or [via `Krew`](https://github.com/kubernetes-sigs/krew) on any OS.

Example via Krew:

```bash
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

The [Helm documentation](https://docs.helm.sh/) is quite good, but this section highlights 2 specific features that should be used from the start to create good reusable Charts:

* [Dependencies](https://docs.helm.sh/chart-best-practices/#requirements)
* [Lifecycle Hooks](https://docs.helm.sh/developing-charts/#chart-lifecycle-hooks)

Good examples always help out a lot. Here is a list of existing git Charts repositories:

* [Kubernetes Charts](https://github.com/helm/charts/)
* [Skyscrapers Charts](https://github.com/skyscrapers/charts)

The above Chart repositories contain Charts that serve as building blocks for bigger composite installations.

## Ingress

> [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) is deployed by default.

### HTTP traffic (ports 80 and 443)

At the moment, HTTP (ports 80 and 443) ingress to the cluster is done as follows:

```console
public ELB -> NGINX Ingress -> Pods (through Service selectors)
```

To make your deployment accessible from the outside world through HTTP(S), you need to create an `Ingress` object with the following annotation: `kubernetes.io/ingress.class: "nginx"`. This will tell the Nginx ingress controller to route traffic to your services. You can find more information on how the Nginx ingress controller works and some examples in the official documentation: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/

### Other traffic

If your application needs to be accessible through other ports or through TCP, you'll need to create your own ingresses. Normally you'll want to create a `Service` of type `LoadBalancer`. With that, Kubernetes will automatically create an ELB that will route traffic to your pods on the needed ports. An example of this kind of `Service` can be found here: https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer

## DNS

> [ExternalDNS](https://github.com/kubernetes-incubator/external-dns) is deployed by default.

It will automatically configure DNS records in Route 53 for your application. For your Ingress, it well add records for the `host` field by default.

To add a record to a `Service` object, you can use the following annotation:

```yaml
external-dns.alpha.kubernetes.io/hostname: nginx.example.org.
```

The only requirement for this to work is that the DNS zone for the domain is hosted in Route53 in the same AWS account as the cluster.

## Automatic SSL certificates

> [Cert-manager](https://docs.cert-manager.io/en/latest/) is deployed by default.

As with the DNS records, SSL certificates can also be automatically fetched and setup for applications deployed on the Kubernetes cluster via [`cert-manager`](https://github.com/jetstack/cert-manager/). We deploy a `letsencrypt-prod` [`ClusterIssuer`](https://cert-manager.readthedocs.io/en/latest/reference/clusterissuers.html) by default, which uses `dns01` validation via Route 53 (used in conjunction with ExternalDNS).

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

With the `hosts` array of the `tls` section you're telling the cluster which domains need to be in the certificate.

Of coure you can also define your own [`Issuers`](https://cert-manager.readthedocs.io/en/latest/reference/issuers.html) and/or [`ClusterIssuers`](https://cert-manager.readthedocs.io/en/latest/reference/clusterissuers.html).

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

There are way more possibilities than described in the examples, which you can find in the [official documentation](https://cert-manager.readthedocs.io).

#### Get a LetsEncrypt certificate using defaults (dns01)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
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
  - hosts:
    - foo.staging.skyscrape.rs
    secretName: foo-staging-tls
```

#### Get a LetsEncrypt certificate using the http01 challenge

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/tls-acme: "true"
    certmanager.k8s.io/acme-challenge-type: "http01"
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
  - hosts:
    - bar.staging.skyscrape.rs
    secretName: bar-staging-tls
```

#### Get a LetsEncrypt wildcard certificate the dns01 challenge

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
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
  - hosts:
    - '*.staging.skyscrape.rs'
    secretName: wildcard-staging-skyscrape-rs-tls
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
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
  - hosts:
    - '*.staging.skyscrape.rs'
    secretName: wildcard-staging-skyscrape-rs-tls
```

You could also issue a `Certificate` first to re-use that later in your `Ingresses`:

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: wildcard-staging-skyscrape-rs
  namespace: default
spec:
  secretName: wildcard-staging-skyscrape-rs-tls
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-prod
  commonName: '*.skyscrape.rs'
  dnsNames:
  - skyscrape.rs
  acme:
    config:
    - dns01:
        provider: route53
      domains:
      - '*.skyscrape.rs'
      - skyscrape.rs
```

**Note**: While it is possible to generate multiple wildcard certificates via a different `secretName`, it is advised / more efficient to reuse the same `Secret` for all ingresses using the wildcard.

**Note 2**: A `Secret` is scoped within a single `Namespace`, which means if you want to use a wildcard certificate in another `Namespace` the cert-manager will request and validate a new certificate from LetsEncrypt. (unless you replicate the `Secrets`)

## IAM Roles

> [Kube2iam](https://github.com/jtblin/kube2iam) is deployed by default.

Your deployments can be assigned with specific IAM roles to grant them fine-grained permissions to AWS services. To do that just add the following annotation to `.spec.template.metadata.annotations` in your `Deployment` objects:

```yaml
iam.amazonaws.com/role: arn:aws:iam::123456789:role/kube2iam/something
```

You can find some examples in the `kube2iam` official documentation: <https://github.com/jtblin/kube2iam#kubernetes-annotation>

**Note**: when you create a new IAM role for a Kubernetes deployment, make sure you create it under the `/kube2iam/` path, otherwise your deployments won't be able to assume it. Also you'll need to add a [trust relationship](http://docs.aws.amazon.com/directoryservice/latest/admin-guide/edit_trust.html) to your role so the k8s nodes roles can assume it. Normally the k8s nodes role will have the following name scheme: `arn:aws:iam::<aws-account-id>:role/nodes.<cluster-domain-name>`

## Cronjobs

Kubernetes can run cronjobs for you. More information/examples about cronjobs can be found [here](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).

### Monitoring

Monitoring for cronjobs is implemented by default. This is done with prometheus and will alert when the last run of the cronjob has failed.

The following alerts are covering different failure cases accordigly:  

* KubeJobCompletion: Warnning alert after 1 hour if any *Job* doesn't succeed or doesn't run at all.
* KubeJobFailed: Warning alert after 1 hour if any *Job* failed
* KubeCronJobRunning: Warning alert after 1 hour if a *CronJob* keeps on running

### Clean up

Starting from Kubernetes 1.7 the scheduled jobs don't get automatically cleaned up. So make sure that you add the following two lines to the `spec` section of your cronjob.

```yaml
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 3
```

This will clean up all jobs except the last 3, both for successful and failed jobs.
