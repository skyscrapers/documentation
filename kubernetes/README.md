<!-- markdownlint-disable no-inline-html -->

# Walk-through of the Skyscrapers' Kubernetes cluster

Here you'll find user-level documentation of our Kubernetes reference solution. You'll learn how to access the cluster, how to deploy applications and services and how to monitor them. You'll also find some tips & tricks on how to get the most from your Kubernetes cluster.

The best place to start is this walk-through. After that you can read more in-depth documentation on specific topics in their respective files in this folder.

If you are new to Kubernetes, check the [getting started page](getting_started.md) for more information.

- [Walk-through of the Skyscrapers' Kubernetes cluster](#walk-through-of-the-skyscrapers-kubernetes-cluster)
  - [Requirements](#requirements)
  - [Authentication](#authentication)
  - [Deploying on Kubernetes with the Helm Package Manager](#deploying-on-kubernetes-with-the-helm-package-manager)
  - [Ingress](#ingress)
    - [HTTP traffic (ports 80 and 443)](#http-traffic-ports-80-and-443)
    - [Other traffic](#other-traffic)
    - [Add authentication with oauth2\_proxy](#add-authentication-with-oauth2_proxy)
    - [Dynamic, whitelabel-style Ingress to your application](#dynamic-whitelabel-style-ingress-to-your-application)
    - [Enabling and using the ModSecurity WAF](#enabling-and-using-the-modsecurity-waf)
  - [DNS](#dns)
  - [Automatic SSL certificates](#automatic-ssl-certificates)
    - [Examples](#examples)
      - [Get a LetsEncrypt certificate using defaults (dns01)](#get-a-letsencrypt-certificate-using-defaults-dns01)
      - [Get a LetsEncrypt certificate using the http01 challenge](#get-a-letsencrypt-certificate-using-the-http01-challenge)
      - [Get a LetsEncrypt wildcard certificate the dns01 challenge](#get-a-letsencrypt-wildcard-certificate-the-dns01-challenge)
    - [Get a certificate for a delegated domain name](#get-a-certificate-for-a-delegated-domain-name)
  - [IAM Roles](#iam-roles)
  - [Storage](#storage)
    - [Persistent Volumes](#persistent-volumes)
    - [Local NVMe Instance Storage](#local-nvme-instance-storage)
  - [Monitoring](#monitoring)
  - [Logs](#logs)
  - [Cluster updates and rollouts](#cluster-updates-and-rollouts)
  - [Cronjobs](#cronjobs)
    - [Cronjob Monitoring](#cronjob-monitoring)
    - [Clean up](#clean-up)
  - [Accessing cluster resources and services locally](#accessing-cluster-resources-and-services-locally)

## Requirements

- [kubectl 1.32.x (1.32.1)](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.32.md#v1321)
- [helm >= 3.15.x (3.15.2)](https://github.com/helm/helm/releases/tag/v3.15.2)
- [AWS cli >= 2.13.0](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

Optionally you can install some other tools to make your life easier. These are documented in the [tools page](tools.md).

## Authentication

To gain access to an EKS cluster you need to authenticate via AWS IAM and configure your kubeconfig accordingly. To do this you'll need a recent version of `awscli` (`>= 2.13.0`). If you don't have the AWS CLI yet, you can install it by [following the AWS instructions](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) or via [Homebrew/Linuxbrew](https://brew.sh/):

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

## Deploying on Kubernetes with the Helm Package Manager

After a roll out of a Kubernetes cluster, it could be tempting to start executing numerous `kubectl create` or `kubectl apply` commands to get stuff deployed on the cluster.

Running such commands is a good idea to learn how deployments are done on Kubernetes, but it **is not the appropriate tool** to construct fully self contained application deployments. The Kubernetes community came up with a separate tool for that:

[The Helm Package Manager](https://helm.sh/)

With Helm, you create self contained packages for a specific piece of a deployment, e.g. a web stack. Such packages are called `Charts` in Helm terminology.

You probably want such a stack to be deployed in a specific Kubernetes namespace, with a specific configuration (`ConfigMap`), defining a Kubernetes `Service` referring to a `Deployment`. But if you want to have this setup reproducible, you need a way to parameterize this.
By using a Template engine, a [Go function library](http://masterminds.github.io/sprig/) and the use of a `Values.yaml` file, you can build a template of a specific piece and re-use that for multiple deployments.

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

To make your deployment accessible from the outside world through HTTP(S), you need to create an `Ingress` object with the following `ingressClassName`: `nginx`. This will tell the Nginx ingress controller to route traffic to your services. You can find more information on how the Nginx ingress controller works and some examples in the [official documentation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/).

To use the internal-only Ingress, you need to set `ingressClassName: nginx-internal` on your `Ingress`.

### Other traffic

If your application needs to be accessible through other ports or through TCP, you'll need to create your own ingresses. Normally you'll want to create a `Service` of type `LoadBalancer`. With that, Kubernetes will automatically create an ELB that will route traffic to your pods on the needed ports. An example of this kind of `Service` can be found [here](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer).

*Please note that launching extra ELBs increases your AWS costs.*

### Add authentication with oauth2_proxy

We use [OAuth2-Proxy and Dex](/kubernetes/authentication.md#accessing-our-various-dashboards-dex), to authenticate web requests to internal resources such as Prometheus, Alertmanager, ... You can also make use of this feature to provide an OAuth2 layer in front of your (internal) applications by setting up NGINX ingress with [External OAUTH Authentication](https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/)

The NGINX Ingress will pass requests through the `oauth2_proxy` to provide the proper Authentication. It receives all traffic from the nginx-ingress, checks if a an `_oauth2_proxy` cookie exists and verifies is. If it doesn't exist the user is redirected to Dex for authentication. You need to set the following annotations on the Ingress to authenticate

```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-url: "https://${oauth2_proxy_domain_name}/oauth2/auth"
  nginx.ingress.kubernetes.io/auth-signin: "https://${oauth2_proxy_domain_name}/oauth2/start?rd=https://$host$request_uri$is_args$args"

# Where oauth2_proxy_domain_name = login.$CLUSTER_NAME, eg. login.staging.eks.example.com
```

It is also possible to limit access to only certain groups by spcifying them as follows (make sure to urlencode them):

```yaml
## Allow groups foo:bar and foo:ipsum access (urlencoded)
annotations:
  nginx.ingress.kubernetes.io/auth-url: "https://${oauth2_proxy_domain_name}/oauth2/auth?allowed_groups=foo%3Abar,foo%3Aipsum"
  nginx.ingress.kubernetes.io/auth-signin: "https://${oauth2_proxy_domain_name}/oauth2/start?rd=https://$host$request_uri$is_args$args"
```

It's important that these groups also need to be whitelisted in the central Dex configuration so they can be passed through to the oauth2_proxy.

If the cookie returned has the correct credentials, the user's request is passed through to the backend. Depending on what the backend uses for Authorisation, you can pass the proper cookies or headers via a `configuration-snippet` annotation. For example to pass the user's JWT bearer token to the backend:

```yaml
annotations:
  nginx.ingress.kubernetes.io/configuration-snippet: |
    auth_request_set $token $upstream_http_authorization;
    proxy_set_header Authorization $token;
```

Or if you jusrt want to pass the authenticated user's email address:

```yaml
annotations:
  nginx.ingress.kubernetes.io/configuration-snippet: |
    auth_request_set $email $upstream_http_x_auth_request_email;
    proxy_set_header X-WEBAUTH-USER $email;
```

If the user has previously [logged in through Dex](/kubernetes/authentication.md#accessing-our-various-dashboards-dex), the flow is fully transparent to the user.

### Dynamic, whitelabel-style Ingress to your application

If your application allows for end-customers to use their custom domain, you can let your application interface directly with the K8s API to manage `Ingress` objects. For more info, check our [separate page on the subject](./create_ingress_via_api.md).

### Enabling and using the ModSecurity WAF

*Note that this is still just some very basic info on enabling the ModSecurity engine in the Ingress controller. More info on usage, fixing false positives etc might be added later.*

If you want to use ModSecurity, first ask your Lead to enable this optional feature via the cluster definition file:

```yaml
tfvars:
  addons:
    nginx_controller_additional_configs: |
      enable-modsecurity: "true"
```

Once enabled cluster-wide, you can enable and finetune ModSecurity through your [Ingress' annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#modsecurity). The [upstream documentation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#modsecurity) isn't very clear on this, but these are our findings which we verified to work.

To just enable ModSecurity in detection-only mode using the [recommended configuration](https://github.com/SpiderLabs/ModSecurity/blob/v3/master/modsecurity.conf-recommended), set the following annotation:

```yaml
nginx.ingress.kubernetes.io/enable-modsecurity: "true"
```

To enable the [OWASP Core Rule Set](https://www.modsecurity.org/CRS/Documentation/) and enable ModSecurity in enforcing mode, set the following annotation:

```yaml
nginx.ingress.kubernetes.io/modsecurity-snippet: |
  SecRuleEngine On
  Include /etc/nginx/owasp-modsecurity-crs/nginx-modsecurity.conf
```

Through this `nginx.ingress.kubernetes.io/modsecurity-snippet` annotation you can also further configure and finetune the ModSecurity configuration. Please consult the [ModSecurity](https://www.modsecurity.org/documentation.html) & [OWASP CRS](https://www.modsecurity.org/CRS/Documentation/) documentation for more information.

## DNS

> [ExternalDNS](https://github.com/kubernetes-incubator/external-dns) is deployed by default.

It will automatically configure DNS records in Route 53 for your application. For your Ingress, it well add records for the `host` field by default.

To exclude an `Ingress` from being managed by external-dns, you can use the following annotation:

```yaml
external-dns.alpha.kubernetes.io/exclude: "true"
```

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

With the `hosts` array of the `tls` section you're telling `cert-manager` which domains need to be in the certificate, and in which `Secret` it should store the private key.

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

`cert-manager` can also issue certificates for delegated domains. If the domain name you want to use for your ingress is hosted in some external DNS servers where `cert-manager` doesn't have access, you can delegate the ACME validation domain to the cluster DNS zone by creating a CNAME record in the external DNS servers. You can read more about this feature in the [official `cert-manager` documentation](https://cert-manager.io/docs/configuration/acme/dns01/#delegated-domains-for-dns01) and in the [example below](#get-a-certificate-for-a-delegated-domain-name).

> [!WARNING]
> [Cert Manager has a limit of 60 parallel certificate Challenges that can be processed at the same time](https://cert-manager.io/docs/concepts/acme-orders-challenges/#challenge-scheduling). If this limit has been reached, and it's filled with unprocessable Challenges (eg. due to DNS misconfiguration / not propagated), then these will block any further certificate issuance. You can see how many Challenges are currently being processed by running `kubectl get challenges -A` (check / count which are `pending`).

### Examples

Below are some simple examples on how to issue certicicates as usually done on the `Ingress`.

There are way more possibilities than described in the examples, which you can find in the [official documentation](https://cert-manager.io/docs/).

#### Get a LetsEncrypt certificate using defaults (dns01)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/tls-acme: "true"
  name: foo
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: foo.staging.skyscrape.rs
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: foo
                port:
                  name: http
  tls:
    - secretName: foo-staging-tls
      hosts:
        - foo.staging.skyscrape.rs
```

#### Get a LetsEncrypt certificate using the http01 challenge

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/tls-acme: "true"
  labels:
    use-http-solver: "true"
  name: bar
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: bar.staging.skyscrape.rs
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: bar
                port:
                  name: http
  tls:
    - secretName: bar-staging-tls
      hosts:
        - bar.staging.skyscrape.rs
```

#### Get a LetsEncrypt wildcard certificate the dns01 challenge

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/tls-acme: "true"
  name: lorem
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: lorem.staging.skyscrape.rs
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: lorem
                port:
                  name: http
  tls:
    - secretName: wildcard-staging-skyscrape-rs-tls
      hosts:
        - '*.staging.skyscrape.rs'
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/tls-acme: "true"
  name: ipsum
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: ipsum.staging.skyscrape.rs
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ipsum
                port:
                  name: http
  tls:
    - secretName: wildcard-staging-skyscrape-rs-tls
      hosts:
        - '*.staging.skyscrape.rs'
```

You could also issue a `Certificate` first to re-use that later in your `Ingresses`:

```yaml
apiVersion: cert-manager.io/v1
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

> [!NOTE]
> While it is possible to generate multiple wildcard certificates via a different `secretName`, it is advised / more efficient to reuse the same `Secret` for all ingresses using the wildcard.
> [!NOTE]
> A `Secret` is scoped within a single `Namespace`, which means if you want to use a wildcard certificate in another `Namespace` cert-manager will request and validate a new certificate from LetsEncrypt (unless you replicate the `Secrets`).

### Get a certificate for a delegated domain name

Let's say you want a certificate for the domain `api.example.com` but `cert-manager` doesn't have access to the root DNS zone for `example.com`. In this situation you can create a `CNAME` record in the `example.com` DNS zone like this:

```bash
_acme-challenge.api.example.com   IN   CNAME   _acme-challenge.api.production.eks.example.org.
```

*Considering `production.eks.example.org` is the FQDN of your K8s cluster.

After this, you create your `Ingress` as described in the examples above, and `cert-manager` will follow the CNAME record to create the certificate validation record in the cluster DNS zone, where it does have access.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/tls-acme: "true"
  name: api
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  name: http
  tls:
    - secretName: api-example-com-tls
      hosts:
        - api.example.com
```

## IAM Roles

> [EKS IAM roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) is used by default.

Your deployments can be assigned with specific IAM roles to grant them fine-grained permissions to AWS services. To do that you'll need to create a [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) for your Pod and annotate it with the IAM role to use. For example

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::889180461196:role/kube/staging-eks-example-com-myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels: {}
spec:
  selector:
    matchLabels: {}
  template:
    metadata: {}
    spec:
      serviceAccountName: myapp
      # Important to set correct fsGroup, depending on which user your app is
      # running as.
      # https://github.com/aws/amazon-eks-pod-identity-webhook/issues/8
      securityContext:
        fsGroup: 1001
        # For completeness you can add the following too (not required)
        #runAsNonRoot: true
        #runAsGroup: 1001
        #runAsUser: 1001
```

It's important to **use [a recent AWS SDK](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html)** in your application for IRSA support.

You can find more examples and technical documentation in the official documentation: <https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html>

For JAVA-based applications IRSA does not work out of the box, you need to do the following change for your application, quoting from <https://pablissimo.com/1068/getting-your-eks-pod-to-assume-an-iam-role-using-irsa> :
> you need to add an instance of `STSAssumeRoleWithWebIdentitySessionCredentialsProvider` to a credentials chain, and pass that custom chain to your SDK init code via the `withCredentials` builder method.
This class doesn’t automatically come as part of the credentials chain. Nor does it automatically initialise itself from environment variables the same way other providers do.
You’ll have to pass in the web identity token file, region name and role ARN to get it running
> [!NOTE]
> Usually a Skyscrapers engineer will create the required IAM roles and policies for you. It's important that we match your ServiceAccount to the IAM policy's `Condition`. If you manage these policies yourself, it's important to setup the IAM role with the correct federated trust relationship. For example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:${namespace}:${service-account-name}"
        }
      }
    }
  ]
}
```

## Storage

### Persistent Volumes

[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) in our cluster are backed by AWS EBS volumes. Among the obvious caveats around scheduling (a volume is limited to a single AZ), there's also a more silent and hard to predict caveat.

[Depending on EC2 instance type, most of them support a maximum of only 28 attachments, including network interfaces, EBS volumes, and NVMe instance store volumes.](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/volume_limits.html#instance-type-volume-limits). This means that only a limited number of EBS volumes per K8s node can be used, also considering our CNI uses multiple network interfaces.

Kubernetes [limits the max amount of volumes for M5,C5,R5,T3 and Z1D to only 25 volumes to be attached to a Node](https://kubernetes.io/docs/concepts/storage/storage-limits/#dynamic-volume-limits), however this often isn't enough depending how much network interfaces are in use by the CNI.

Unfortunately AWS doesn't throw an error either when this happens. Instead a Volume will stay stuck in the `Attaching` state and your Pod will fail to launch. After ~5 minutes Kubernetes will [taint the node](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) with `NodeWithImpairedVolumes`.

We have added a Prometheus alert to catch this taint and you can [follow the actions described in the runbook](https://github.com/skyscrapers/documentation/tree/master/runbook.md#alert-name-nodewithimpairedvolumes) when this happens.

### Local NVMe Instance Storage

Certain AWS EC2 instances come with fast [local NVMe Instance Storage](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html) and can usually be recognized with the `d` suffix (eg. `m5d.large`). Our platform will automatically mount these volumes under the `/ephemeralX` paths (eg. `/ephemeral0`, `/ephemeral1`, ...).

You can use this storage by mounting it via a [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) volume in your Pod spec, for example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
    - name: test
      image: k8s.gcr.io/test-webserver
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
      volumeMounts:
        - name: ephemeral
          mountPath: /fastdata
          subPathExpr: $(POD_NAME)
  volumes:
    - name: ephemeral
      hostPath:
        path: /ephemeral0
```

It is important to note here that in the example we use the [K8s Downward API](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/) with [subPath expansion](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath-expanded-environment) so each Pod uses it's own subfolder under the `/ephemeral0` path.

## Monitoring

Cluster and application monitoring is a quite extensive topic by itself, so there's a specific document for it [here](./monitoring.md).

## Logs

Cluster and application logging is a quite extensive topic by itself, so there's a specific document for it [here](./logging.md).

## Cluster updates and rollouts

As part of our responsibilities, we continuously roll improvements (upgrades, updates, bug fixes and new features). Depending on the type of improvement, the impact on platform usage and application varies anywhere between nothing to having (a small) downtime. Below an overview for the most common types of improvements. More exceptional types will be handled separately.

|          Type of improvement          | Description                                                                                                                                                                                                                                                                                               |                            Expected impact on your workloads                            |
| :-----------------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------: |
|        Add-ons (non-breaking)         | Improvements expected to not have an impact on the current usage of the cluster or application behaviour.<br/><br/>These are rolled out automatically at any time during the day. You are informed during the updates.                                                                                    |                                        No impact                                        |
| Add-ons (non-breaking but disruptive) | Improvements to add-ons that may lead to temporary unavailability of platform functionalities (monitoring, logging, dashboard, etc) but that do not impact application workloads.<br/><br/>These are rolled out automatically at any time during the day. You are informed before and during the updates. |                                        No impact                                        |
|          Add-ons (breaking)           | These improvements may need changes or intervention by you before they can be rolled out.<br/><br/>We will reach out to you to discuss what’s needed on how the improvement will be rolled out.                                                                                                           |                         In some cases: minimal planned downtime                         |
|         Cluster improvements          | Low-frequency improvements to the foundations of the cluster. Usually these involve rolling updates leading to nodes being recycled.<br/><br/>These are rolled out automatically at any time during the day. You are informed before and during the updates.                                              | Cluster-aware workloads: No impact<br/><br/>Other workloads: potential minimal downtime |

To minimize the impact on your workloads, we suggest you to implement cluster-aware workloads as much as possible (TODO: define cluster-aware) and implement `PodDisruptionBudgets`. There's more information on this [here](./pods.md).

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

## Accessing cluster resources and services locally

One of the main challenges developers and operators face when using Kubernetes, is communication between cluster services and those running in a local workstation. This is sometimes needed to test new versions of a service for example, or to access a cluster service that's not exposed to the internet.

There are multiple solutions to overcome this, depending on the use-case and the specific requirements, although a nice all-round tool that covers most of the use-cases is [Telepresence](https://www.telepresence.io/).

Telepresence creates a proxy tunnel to a Kubernetes cluster, allowing you to directly communicate with cluster services and Pods as if they were running in your local network. Head over to [the documentation](https://www.telepresence.io/docs/latest/howtos/intercepts/) to know more on how it works and how to use it.

Telepresence works out of the box with our managed Kubernetes clusters that are not behind VPN and you can start using it right away on such clusters. For those private clusters that are behind OpenVPN, there's an issue affecting DNS resolution when using Telepresence. We're looking into that issue and we'll update this documentation once we find a solution for it.

Note that the first time Telepresence is used on a cluster, it will automatically install the required cluster components, this requires permissions for creating Namespaces, ServiceAccounts, ClusterRoles, ClusterRoleBindings, Secrets, Services, MutatingWebhookConfiguration, and for creating the `traffic-manager` deployment which is typically done by a full cluster administrator. After that initial setup, these components will keep running in the cluster for future Telepresence usage. A user running Telepresence is expected to only have the minimum cluster permissions necessary to create a Telepresence intercept, and otherwise be unable to affect Kubernetes resources.

If you have trouble running Telepresence for the first time, please contact your Customer Lead or a colleague that has the necessary permissions.
