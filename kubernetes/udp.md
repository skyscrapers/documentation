# Exposing UDP services

It is possible to expose UDP services, however [with some caveats](https://github.com/kubernetes/kubernetes/issues/79523#issuecomment-643610747):

- You need to create a Service of `type=LoadBalancer`. This will create an (extra) AWS Load Balancer, which [involves a cost](https://aws.amazon.com/elasticloadbalancing/pricing/).
- You can not mix TCP and UDP ports within the same Service, so you'll need to split this into 2 separate Services (and thus will also result in 2 separate Load Balancers). For more info, see <https://github.com/kubernetes/enhancements/issues/1435>.
- If you're setting `externalTrafficPolicy: Cluster` on the Service's spec, you'll also need to implement a TCP-based health check as an AWS NLB only supports health checking over TCP. However, this isn't a problem when using `externalTrafficPolicy: Local`.

Simple example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: udp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: udp-test
  namespace: udp
  labels:
    app.kubernetes.io/name: udp-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: udp-test
  template:
    metadata:
      labels:
        app.kubernetes.io/name: udp-test
    spec:
      containers:
        - name: udp
          image: cilium/echoserver-udp
          ports:
            - name: udp-1
              containerPort: 69
              protocol: UDP
---
apiVersion: v1
kind: Service
metadata:
  name: udp-test
  namespace: udp
  labels:
    app.kubernetes.io/name: udp-test
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app.kubernetes.io/name: udp-test
  ports:
    - protocol: UDP
      port: 6969
      targetPort: udp-1
```
