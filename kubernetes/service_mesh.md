# Service Mesh

We use [linkerd2](https://linkerd.io/2/overview) as service mesh. We offer this as an optional feature, so let us know if you want to enable it. You can check if linkerd2 is installed in your cluster with the following command:

```sh
kubectl get pods -n linkerd
```

## Services

To add a service to the mesh, a sidecar, init container and some labels need to be added to the deployment. On the initContainer you have the ability to ignore some incoming or outgoing ports for the deployment, excluding them from the service mesh. This **must be used** for outgoing MongoDB, MySQL, ... connections for example. See the [Protocol support documentation](https://linkerd.io/2/adding-your-service/#server-speaks-first-protocols) for more info.

### labels

```yaml
spec:
  template:
    metadata:
      annotations:
        linkerd.io/created-by: linkerd/cli stable-2.1.0
        linkerd.io/proxy-version: stable-2.1.0
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
      - name: linkerd-proxy
        image: gcr.io/linkerd-io/proxy:stable-2.1.0
        imagePullPolicy: IfNotPresent
        env:
        - name: LINKERD2_PROXY_LOG
          value: warn,linkerd2_proxy=info
        - name: LINKERD2_PROXY_BIND_TIMEOUT
          value: 10s
        - name: LINKERD2_PROXY_CONTROL_URL
          value: tcp://linkerd-proxy-api.linkerd.svc.cluster.local:8086
        - name: LINKERD2_PROXY_CONTROL_LISTENER
          value: tcp://0.0.0.0:4190
        - name: LINKERD2_PROXY_METRICS_LISTENER
          value: tcp://0.0.0.0:4191
        - name: LINKERD2_PROXY_OUTBOUND_LISTENER
          value: tcp://127.0.0.1:4140
        - name: LINKERD2_PROXY_INBOUND_LISTENER
          value: tcp://0.0.0.0:4143
        - name: LINKERD2_PROXY_DESTINATION_PROFILE_SUFFIXES
          value: .
        - name: LINKERD2_PROXY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        livenessProbe:
          httpGet:
            path: /metrics
            port: 4191
          initialDelaySeconds: 10
        ports:
        - containerPort: 4143
          name: linkerd-proxy
        - containerPort: 4191
          name: linkerd-metrics
        readinessProbe:
          httpGet:
            path: /metrics
            port: 4191
          initialDelaySeconds: 10
        resources: {}
        securityContext:
          runAsUser: 2102
        terminationMessagePolicy: FallbackToLogsOnError
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
        image: gcr.io/linkerd-io/proxy-init:stable-2.1.0
        imagePullPolicy: IfNotPresent
        name: linkerd-init
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
          privileged: false
        terminationMessagePolicy: FallbackToLogsOnError
```

**Note:** you can get these deployment changes by running `linkerd inject deployment.yaml` where `deployment.yaml` is one of your existing Deployment specs. Hopefully a [future version will handle injection automatically](https://github.com/linkerd/linkerd2/issues/561).

**Note 2:** if you're not using the `linkerd inject` command and you're adding the sidecar and init containers via Helm for example, make sure you're using the same linkerd version that's deployed in your cluster. You can check such version by running `linkerd version`.

## Other useful commands

```sh
linkerd dashboard
```

```sh
linkerd dashboard --show url
```

```sh
linkerd stat deployments
```
