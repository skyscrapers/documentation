# Service Mesh

- [Service Mesh](#service-mesh)
  - [Istio](#istio)
  - [Add a service to the mesh](#add-a-service-to-the-mesh)
  - [Istio gateways](#istio-gateways)
    - [Cert-manager integration](#cert-manager-integration)
    - [External dns integration](#external-dns-integration)
    - [TLS ciphers](#tls-ciphers)
  - [Kiali](#kiali)

## Istio

We use [Istio](https://istio.io/) as service mesh. We offer this as an optional feature, so let us know if you want to enable it. You can check if Istio is installed in your cluster with the following command:

```sh
kubectl get pods -n istio-system
```

We also deploy [Kiali](https://kiali.io/), together with Istio. Kiali is a web UI dashboard to visualize all sorts of data from the service mesh, as well as manage some of its configuration.

To work with istio and Kiali, it's recommended that you [set up `istioctl`](https://istio.io/latest/docs/setup/getting-started/#download) on your workstation.

## Add a service to the mesh

To add a service into the Istio mesh, you need to add the proxy side-car container to your Pod(s). To make things easy, Istio can automatically inject that container when your Pods are created. To enable that feature for a namespace you just need to label it with the `istio.io/rev=prod` label:

```console
kubectl label ns yournamespace istio.io/rev=prod
```

*Replace `yournamespace` with the actual namespace name where your workload will be running in.*

This will instruct Istio to inject the sidecar container to all Pods in that namespace by default. Alternatively, you can also control the sidecar injection at the Pod level, by setting the `sidecar.istio.io/inject` annotation in the Pod template spec to either `true` or `false`. You can read more about the sidecar injection in the [Istio documentation](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/).

**Note** that you'll still need to trigger a re-deploy of your Pods after the appropriate namespace label and Pod annotations are set, as the Istio mutating webhook admission controller only injects the sidecar container when the Pods are being created.

You can run the following command to check if everything is correctly configured in your namespace to work with Istio:

```console
istioctl analyze --timeout 60s # The timeout is sometimes necessary as it might be a bit slow to run and return a response
```

## Istio gateways

[Istio Gateways](https://istio.io/latest/docs/concepts/traffic-management/#gateways) provide endpoints for your services inside the mesh to the outside world. They work in a similar way as an ingress controller.

Right now we only configure an ingress gateway controller for each cluster, to provide a way for your services to be reachable from the outside world. You can create multiple `Gateway` resources to configure your required entrypoints, and all of them must point to the same controller:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: example
spec:
  selector:
    app: ingressgateway # This selects the ingress gateway controller running in the istio-system namespace
  ...
```

You can find more information with examples on how to work with gateways in the [official documentation](https://istio.io/latest/docs/concepts/traffic-management/#gateways).

### Cert-manager integration

[Similar to how the Nginx ingress controller works](README.md#automatic-ssl-certificates), Istio Gateways can also be configured with TLS certificates provided by cert-manager. In this case though, cert-manager won't automatically create the `Certificate` resource, so that will need to be provided by the user. Here's an example:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: bookinfo-example-com
  namespace: istio-system # This needs to be the namespace where the ingress gateway controller runs
spec:
  secretName: bookinfo-example-com-cert
  commonName: bookinfo.example.com
  dnsNames:
    - bookinfo.example.com
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-prod
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - bookinfo.example.com
      tls:
        mode: SIMPLE
        credentialName: bookinfo-example-com-cert # This needs to match the secretName on the Certificate
```

### External dns integration

[External-dns](README.md#dns) will automatically pick up the host names of Istio Gateways and configure the correct DNS entries if possible (that is if the name servers for those domain names are hosted in Route53 in the same AWS account as the EKS cluster).

### TLS ciphers

A word on TLS ciphers when terminating TLS connections in an Istio Gateway: if you want to support older browsers with legacy SSL libraries and ciphers, you might need to configure custom ciphers in your `Gateway` resources. We've had successful results with the following ciphers:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: foo
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - foo.example.com
      tls:
        cipherSuites:
          - CHACHA20-POLY1305-SHA256
          - ECDHE-ECDSA-AES256-GCM-SHA384
          - ECDHE-ECDSA-AES128-GCM-SHA256
          - ECDHE-RSA-AES128-GCM-SHA256
          - ECDHE-ECDSA-AES256-SHA384
          - ECDHE-ECDSA-AES256-SHA256
          - ECDHE-RSA-AES256-SHA256
          - ECDHE-ECDSA-AES256-SHA
          - ECDHE-RSA-AES256-SHA
          - ECDHE-RSA-AES256-GCM-SHA384
          - AES256-GCM-SHA384
          - AES128-GCM-SHA256
```

You can run your endpoints through an SSL analyzer tool like [SSL Labs](https://www.ssllabs.com/ssltest/index.html) to verify SSL support from different platforms and browsers.

## Kiali

Kiali dashboard is not publicly accessible, to be able to access it you must run the following command in your terminal:

```console
istioctl dashboard kiali
```
