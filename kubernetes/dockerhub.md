# Dockerhub

Regarding the [latest changes](https://docs.docker.com/docker-hub/usage/) with DockerHub, starting March 1 2025, the throttling limits for unauthenticated calls are reduced to 10 pulls per hour per IP address.

Best way to keep using DockerHub is to authenticate and use your credentials.

The steps are:

1. Register on [docker.com](https://www.docker.com) for a Pro/Team/Business plan depending on your needs
2. Create a new secret that includes your dockerhub credentials ([upstream docs](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line)):
  - `kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>`
  - use `https://index.docker.io/v1/` as a value of the `--docker-server` if you're authenticating against dockerhub
3. Ensure this secret is created in every namespace that needs to pull an image from the private registry.
4. In your container specification, you will use the following `imagePullSecret` [upstream docs](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret):
```yaml
spec:
  containers:
  - name: <your-container-name>
    image: <the-image-from-dockerhub>
  imagePullSecrets:
  - name: regcred # Or whatever you decided to name your secret from step 2
```
4. Congratulations, now you're authenticated to DockerHub

In the future, you can also consider to use `ECR` Pull Through Cache, which makes you mitigate the authentication limits even further, by pulling once if an image has never been pulled before, and the consequent pulls will be coming from the cache, so it doesn't go over even to dockerhub.
