# Service Mesh

We use [linkerd2](https://linkerd.io/2/overview) for service mesh. It might not be installed in your cluster by default, as we only set it up on demand. You can check if linkerd2 is installed in your cluster with the following command:

```
kubectl get pods -n linkerd
```

## Services

To add a service to the mesh, a sidecar, init container and some labels need to be added to the deployment. On the initContainer you have the ability to ignore some incoming or outgoing ports for the deployment, excluding them from the service mesh. This can be used for outgoing MongoDB, MySQL, ... connections for example. See the [Protocol support documentation](https://linkerd.io/2/adding-your-service/#server-speaks-first-protocols) for more info.

### labels

```yaml
spec:
  template:
    metadata:
      annotations:
        linkerd.io/created-by: linkerd/cli v18.8.1
        linkerd.io/proxy-version: v18.8.1
      labels:
        linkerd.io/control-plane-ns: linkerd
        linkerd.io/proxy-deployment: deploymentName
```

### sidecar

```yaml
spec:
  template:
    spec:
      containers:
      - env:
        - name: LINKERD2_PROXY_LOG
          value: warn,linkerd2_proxy=info
        - name: LINKERD2_PROXY_BIND_TIMEOUT
          value: 10s
        - name: LINKERD2_PROXY_CONTROL_URL
          value: tcp://proxy-api.linkerd.svc.cluster.local:8086
        - name: LINKERD2_PROXY_CONTROL_LISTENER
          value: tcp://0.0.0.0:4190
        - name: LINKERD2_PROXY_METRICS_LISTENER
          value: tcp://0.0.0.0:4191
        - name: LINKERD2_PROXY_PRIVATE_LISTENER
          value: tcp://127.0.0.1:4140
        - name: LINKERD2_PROXY_PUBLIC_LISTENER
          value: tcp://0.0.0.0:4143
        - name: LINKERD2_PROXY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: gcr.io/linkerd-io/proxy:v18.8.1
        imagePullPolicy: IfNotPresent
        name: linkerd-proxy
        ports:
        - containerPort: 4143
          name: linkerd-proxy
        - containerPort: 4191
          name: linkerd-metrics
        resources: {}
        securityContext:
          runAsUser: 2102
```

### initContainer

```yaml
spec:
  template:
    spec:
      initContainers:
      - args:
        - --incoming-proxy-port
        - "4143"
        - --outgoing-proxy-port
        - "4140"
        - --proxy-uid
        - "2102"
        - --inbound-ports-to-ignore
        - 4190,4191
        image: gcr.io/linkerd-io/proxy-init:v18.8.1
        imagePullPolicy: IfNotPresent
        name: linkerd-init
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
          privileged: false
```

**Note:** you can get these deployment changes by running `linkerd inject deployment.yaml` where `deployment.yaml` is one of your existing Deployment specs. Hopefully a [future version will handle injection automatically](https://github.com/linkerd/linkerd2/issues/561).

## Other useful commands

`linkerd dashboard`
`linkerd dashboard --show url`
`linkerd stat deployments`
