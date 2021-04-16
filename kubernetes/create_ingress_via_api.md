# Dynamic, whitelabel-style Ingress to your application

If your application allows for end-customers to use their custom domain, we've [offered Caddy](https://changelog.skyscrapers.eu/general/2020/03/23/caddy-1.0.4.html) to provide on-demand SSL certificates for a while.

However with our Kubernetes reference solution there is a more native and scalable solution through the Nginx Ingress Controller and Cert-manager.

Our recommended way is that [your application interacts directly with the Kubernetes API](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod) to register (and de-register) Ingress objects on-demand. You can find a list of [official and community-supported K8s client libraries in the documentation](https://kubernetes.io/docs/reference/using-api/client-libraries/).

Via the API you can submit an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) for the end-customer's domain, for example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
    kubernetes.io/tls-acme: "true"
  labels:
    use-http-solver: "true"
  name: www-example-com
  namespace: my-namespace
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  name: http
  tls:
    - secretName: www-example-com-tls
      hosts:
        - www.example.com
```

Here's an example of how to create such Ingress via the Node.js K8s library: <https://github.com/kubernetes-client/javascript/blob/master/examples/ingress.js>.

Of course your application will also need credentials and the correct permissions to allow interaction with the K8s API. This can be controlled through [Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) and [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

Create a `ServiceAccount`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: my-namespace
```

Create a `Role` with the proper permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app
  namespace: my-namespace
rules:
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Create a `RoleBinding` to attach the `Role` to the `ServiceAccount`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-app
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: my-namespace
```

Set the `ServiceAccount` for your `Pod`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-namespace
spec:
  template:
    metadata:
    # ...
    spec:
      serviceAccountName: my-app
      containers:
      # ...
```

This will automount the service account's `Secret` in your Pod and you can extract the required credentials from the following location:

- Token: `/var/run/secrets/kubernetes.io/serviceaccount/token`
- CA cert: `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`

As endpoint of the API server it is recommended to use `kubernetes.default.svc`.

**Note**: Most client libraries will automatically pick up these credentials.

For more information on accessing the API from a Pod, and an example using the Go client library, check out the official documentation: <https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod>.
