# Dockerhub

Regarding the [latest changes](https://docs.docker.com/docker-hub/usage/) with DockerHub, starting March 1 2025, the throttling limits for unauthenticated calls are reduced to 10 pulls per hour per IP address.

## Alternatives

Internally, we use the following process to mitigate the DockerHub rate limits. In short, we first try to replace the image with an official or mirrored alternative. If that fails, we authenticate to DockerHub with a Pro account.

1. Search for **official** alternatives of the image on [`quay.io`](https://quay.io/search), `ghcr.io`, `gcr.io`, ...
2. Search for a **Verified Account** mirror on [`public.ecr.aws`](https://gallery.ecr.aws/search)
3. Check whether the `docker.io` (DockerHub) image is from a "Verified Publisher", as [no rate limiting should apply according to the docs](https://docs.docker.com/docker-hub/usage/pulls/#pull-attribution)
4. Check whether you can pull the same image from [`mirror.gcr.io`](https://cloud.google.com/artifact-registry/docs/pull-cached-dockerhub-images)
5. Stay on `docker.io` (DockerHub) and use a Pro account with `imagePullSecrets` (read further below)

## DockerHub Authentication

Best way to keep using DockerHub is to authenticate and use your credentials (using [a Docker Pro license](https://www.docker.com/pricing/)).

The steps are:

1. Register on [docker.com](https://www.docker.com) for a Pro/Team/Business plan depending on your needs
2. Create a new secret that includes your dockerhub credentials ([upstream docs](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line)):

    ```shell
    kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
    ```

3. Ensure this secret is created in **every namespace** that needs to pull an image from the private registry.
4. In your container specification, you will use the following `imagePullSecret` [upstream docs](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret):

    ```yaml
    spec:
      containers:
      - name: <your-container-name>
        image: <the-image-from-dockerhub>
      imagePullSecrets:
      - name: regcred # Or whatever you decided to name your secret from step 2
    ```

5. Congratulations, now you're authenticated to DockerHub

Alternatively, you can also do it via helm ([upstream docs](https://helm.sh/docs/howto/charts_tips_and_tricks/#creating-image-pull-secrets)):

1. Have your credentials defined in your `values.yaml`:

    ```yaml
    imageCredentials:
      registry: index.docker.io/v1/
      username: someone
      password: sillyness
      email: someone@host.com
    ```

2. Define our helper template as follows:

    ```helm
    {{- define "imagePullSecret" }}
    {{- with .Values.imageCredentials }}
    {{- printf "{\"auths\":{\"%s\":{\"username\":\"%s\",\"password\":\"%s\",\"email\":\"%s\",\"auth\":\"%s\"}}}" .registry .username .password .email (printf "%s:%s" .username .password | b64enc) | b64enc }}
    {{- end }}
    {{- end }}
    ```

3. Use the helper template in a larger template to create the `Secret` manifest:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: regcred
    type: kubernetes.io/dockerconfigjson
    data:
      .dockerconfigjson: {{ template "imagePullSecret" . }}
    ```

4. Follow step 4 in the instructions above, to ensure your workload is using this `imagePullSecret`

In the future, you can also consider to use `ECR` Pull Through Cache, which makes you mitigate the authentication limits even further, by pulling once if an image has never been pulled before, and the consequent pulls will be coming from the cache, so it doesn't go over even to dockerhub.
