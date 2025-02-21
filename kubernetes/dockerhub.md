# Dockerhub

Regarding the [latest changes](https://docs.docker.com/docker-hub/usage/) with DockerHub, starting March 1 2025, the throttling limits for unauthenticated calls are reduced to 10 pulls per hour per IP address.

Best way to keep using Dockerhub is to authenticate and use your credentials.

The steps are:

1. Register on [docker.com](https://www.docker.com) for a Pro/Team/Business plan
2. Pass us the credentials on a secure medium. We will update your cluster definition file and include the following:
```yaml
infra_registry:
    enabled: true
    server: https://index.docker.io/v1/
    username: <your username from step 1>
    email: <your email>
    password_payload: <we encrypt your password with KMS >
```
3. In your container specification, you will use the following `imagePullSecret` [upstream docs](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret):
```yaml
spec:
  containers:
  - name: <your-container-name>
    image: <the-image-from-dockerhub>
  imagePullSecrets:
  - name: image-pull-secret
```
4. Congratulations, now you're authenticated to Dockerhub

In the future, you can also consider to use `ECR` Pull Through Cache, which makes you mitigate the authentication limits even further, by pulling once if an image has never been pulled before, and the consequent pulls will be coming from the cache, so it doesn't go over even to dockerhub.

> [!NOTE]
> You have to also inform us on which namespaces you would like to expose this secret.
