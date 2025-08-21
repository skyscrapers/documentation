# Concourse

At Skyscrapers we've standardized on Concourse for our CI/CD system, both for ourselves and for our customers. It is an integral part of our latest Kubernetes & AWS ECS Reference Solutions

Concourse is a next generation continuous integration and delivery tool. Similar to configuration management tools, Concourse CI/CD pipelines are described as code and put under source control. This is a major improvement over other CI systems and setups that are often difficult to reproduce in case of failure.

You'll find your Concourse address in the `README.md` file of your GitHub repo. If you don't already have a Concourse setup, ping us and we can set it up for you.

- [Concourse](#concourse)
  - [CLI](#cli)
  - [Authentication](#authentication)
    - [Logging in with fly](#logging-in-with-fly)
  - [Useful commands and procedures](#useful-commands-and-procedures)
  - [Set pipelines](#set-pipelines)
    - [Get the newest fly](#get-the-newest-fly)
    - [Removing a stalled worker](#removing-a-stalled-worker)
  - [Secrets](#secrets)
    - [Kubernetes integration](#kubernetes-integration)
    - [Plain secrets.yaml file](#plain-secretsyaml-file)
  - [Auto-update pipelines](#auto-update-pipelines)
    - [Pipeline automation without a secrets manager](#pipeline-automation-without-a-secrets-manager)
      - [Using Git](#using-git)
      - [Using another Concourse resource](#using-another-concourse-resource)
  - [docker-image deprecation](#docker-image-deprecation)
  - [Adding extra auth credentials for registry-image build](#adding-extra-auth-credentials-for-registry-image-build)
    - [DockerHub](#dockerhub)
    - [ECR](#ecr)
  - [helm v3](#helm-v3)
  - [Feature environments](#feature-environments)

## CLI

Concourse can be accessed either via the web UI or the CLI (`fly`). The web UI is very useful for getting a visual state of the pipelines, as well as triggering and cancelling jobs, but Concourse is primarily driven from the command-line. You can read more about the `fly` CLI in the [official Concourse documentation](https://concourse-ci.org/fly.html).

A bit further in this page you'll also find some highlights and examples of common commands about `fly`.

## Authentication

We plug Concourse into the same Dex IdP we provide for other Dashboards like Grafana etc, so everything should already work. If not, contact us on Slack and we'll help you further.

### Logging in with fly

`fly` can be used to access multiple Concourse servers, to accomplish that it has the concept of targets. Each target represents a different Concourse setup, and you need to provide the alias of such target to all the `fly` commands with the `-t` flag, so it knows which Concourse server to reach. You can choose the name of your targets, it can be an arbitrary string, and it'll only apply to your computer.

```shell
fly -t youralias login --team-name yourcompanyname --concourse-url https://ci.yourcompanyname.com
```

*Note*: from here on you'll need to specify the `youralias` target in each `fly` command with `-t youralias`.

## Useful commands and procedures

## Set pipelines

```shell
fly -t youralias set-pipeline -p <pipeline-name> -c <path-to-the-ci-dir>/pipelines/<pipeline-name>.yaml
```

For example:

```shell
fly -t youralias set-pipeline -p k8s-staging -c ci/pipelines/k8s-staging.yaml
```

### Get the newest fly

Either download the newest binary from your Concourse web UI or execute `fly -t youralias sync`

### Removing a stalled worker

- Show Concourse workers: `fly -t youralias workers`
- Terminate the stalled worker instance (via EC2 dashboard, cli, whatever)
- Prune the worker from Concourse: `fly -t youralias prune-worker -w <worker name>`

## Secrets

You'll sometimes want to set secrets and other sensitive data in your pipelines, like AWS or DockerHub credentials. There are a couple of options to do that, like [integrating](https://concourse-ci.org/creds.html) with Kubernetes or AWS Secrets Manager, or you can use pipeline parameters to keep your sensitive data out from the pipeline definitions.

### Kubernetes integration

We can enable Concourse to [read secrets from the Kubernetes cluster](https://concourse-ci.org/kubernetes-credential-manager.html). This is one of the most straight forward ways to provide secrets to your pipelines. Get in touch with us through Slack or GH issue to get you started.

### Plain secrets.yaml file

Another option for using sensitive data and information in your Concourse pipelines is to use [pipeline `((params))`](https://concourse-ci.org/setting-pipelines.html#pipeline-params). The syntax of the pipeline definition `yaml` is the same as the one for other integrations, with the difference that you'll need to provide the values of those parameters when setting the pipeline in Concourse with `fly -t youralias set-pipeline -p pipeline_name -c pipeline_definition.yaml -l secrets.yaml`. This way secrets are kept separate from your pipeline definitions and aren't committed to source control.

The downside of this approach is that if the values of those parameters change, you'll have to set the pipeline again with the updated data. Another inconvenience is that it's hard to distribute the parameters file (`secrets.yaml`) to the team, specially when it's frequently updated, as it's ignored by Git. We normally store it internally in our shared password manager, so if you need the latest `secrets.yaml` file you can ask someone from Skyscrapers to provide it (`@help`).

## Auto-update pipelines

Concourse pipelines can also be automated through Concourse itself, so you don't need to create and update pipelines manually via `fly` anymore. This is done using the [Concourse pipeline resource](https://github.com/concourse/concourse-pipeline-resource). We highly encourage you to go over the resource documentation to know how to set it up, although we'll provide you with a simple example below.

`mother-of-pipelines.yaml`:

```yaml
---
resource_types:
  - name: concourse-pipeline
    type: docker-image
    source:
      repository: concourse/concourse-pipeline-resource
      tag: latest

resources:
  - name: git-ci
    type: git
    source:
      uri: git@github.com:example/ci.git
      branch: master

  - name: concourse
    type: concourse-pipeline
    source:
      target: https://ci.example.com
      teams:
      - name: yourteamname
        username: concourse
        password: ((CONCOURSE_PASSWORD))

jobs:
  - name: deploy-pipelines
    plan:
      - get: git-ci
        trigger: true
      - put: concourse
        params:
          pipelines_file: git-ci/pipelines-file.yaml
```

`pipelines-file.yaml` being:

```yaml
---
pipelines:
  - name: puppet
    team: yourteamname
    config_file: git-ci/puppet/pipeline.yaml
    unpaused: true
  - name: charts
    team: yourteamname
    config_file: git-ci/charts/pipeline.yaml
    unpaused: true
  - name: docker-images
    team: yourteamname
    config_file: git-ci/docker-images/pipeline.yaml
    unpaused: true
```

With the previous example, Concourse will automatically update all pipelines specified in the `pipelines-file.yaml` file whenever their config files get updated. Also, if you need to add a new pipeline, you just specify it in the `pipelines-file.yaml` file and Concourse will create it for you. Of course, you still need to run `fly set-pipeline` to create the main `mother-of-pipelines` pipeline and keep it up-to-date.

There's one important thing to consider here though: wheather you're using a secrets manager for providing the pipeline `((params))` or not. If you're using a secrets manager, then you don't need to make further changes in your current pipelines, as they'll keep picking up their `((params))` from the secret store. But if you're using an external `params.yaml` or `secrets.yaml` file you might have some extra work ahead. If this is your case, keep reading.

### Pipeline automation without a secrets manager

If you are **not** using a secrets manager (like K8s) for your pipeline `((params))`, you could provide them via one of the following methods.

#### Using Git

Adding the `params.yaml` file in Git, together with your pipeline definitions (**highly discouraged if those parameters contain security sesitive data**). Then it's just a matter of providing that `params.yaml` file via the `vars_files` option.

`pipelines-file.yaml`:

```yaml
---
pipelines:
  - name: puppet
    team: yourteamname
    config_file: git-ci/puppet/pipeline.yaml
    vars_files:
      - git-ci/puppet/params.yaml
    unpaused: true
  - name: charts
    team: yourteamname
    config_file: git-ci/charts/pipeline.yaml
    vars_files:
      - git-ci/charts/params.yaml
    unpaused: true
  - name: docker-images
    team: yourteamname
    config_file: git-ci/docker-images/pipeline.yaml
    vars_files:
      - git-ci/docker-images/params.yaml
    unpaused: true
```

#### Using another Concourse resource

You can also provide the `params.yaml` or `secrets.yaml` files via another Concourse resource, like `s3` for example. This method is more suitable for providing security sensitive data to your pipelines, as the `secrets.yaml` file in S3 can be encrypted and its access restricted.

`mother-of-pipelines.yaml`:

```yaml
---
resource_types:
  - name: concourse-pipeline
    type: docker-image
    source:
      repository: concourse/concourse-pipeline-resource
      tag: latest

resources:
  - name: git-ci
    type: git
    source:
      uri: git@github.com:example/ci.git
      branch: master

  - name: concourse
    type: concourse-pipeline
    source:
      target: https://ci.example.com
      teams:
      - name: yourteamname
        username: concourse
        password: ((CONCOURSE_PASSWORD))

  - name: concourse_secrets
    type: s3
    source:
      bucket: concourse_params
      versioned_file: directory_on_s3/secrets.yaml
      access_key_id: ((ACCESS-KEY))
      secret_access_key: ((SECRET))

jobs:
  - name: deploy-pipelines
    plan:
      - get: git-ci
        trigger: true
      - get: concourse_secrets
        trigger: true
      - put: concourse
        params:
          pipelines_file: git-ci/pipelines-file.yaml
```

`pipelines-file.yaml`

```yaml
---
pipelines:
  - name: puppet
    team: yourteamname
    config_file: git-ci/puppet/pipeline.yaml
    vars_files:
      - concourse_secrets/secrets.yaml
    unpaused: true
  - name: charts
    team: yourteamname
    config_file: git-ci/charts/pipeline.yaml
    vars_files:
      - concourse_secrets/secrets.yaml
    unpaused: true
  - name: docker-images
    team: yourteamname
    config_file: git-ci/docker-images/pipeline.yaml
    vars_files:
      - concourse_secrets/secrets.yaml
    unpaused: true
```

## docker-image deprecation

In the new Concourse 5.0.0 version, a new resource was released to track and upload Docker images to a registry, the [`registry-image-resource`](https://github.com/concourse/registry-image-resource). This new resource is intended to replace the current [`docker-image-resource`](https://github.com/concourse/docker-image-resource), as it's more lightweight and simpler. Concourse announced that they intend to deprecate the current `docker-image-resource` in the future.

The only caveat is that the new `registry-image-resource` doesn't build images, so you need to do that a task before the `put` step.

Example:

```diff
---
resources:
# Git repos
 - name: docker-mongodb-exporter-git
  type: git
  icon: github-circle
  source:
    uri: https://github.com/dcu/mongodb_exporter.git
    branch: master

 - name: mongodb-exporter-image
-  type: docker-image
+  type: registry-image
  icon: docker
  check_every: 24h
  source:
    repository: skyscrapers/mongodb-exporter
    username: ((DOCKER_HUB_SKYSCRAPERS_CREDS.username))
    password: ((DOCKER_HUB_SKYSCRAPERS_CREDS.password))

jobs:
  plan:
    - get: docker-mongodb-exporter-git
      trigger: true
+    - task: docker-build
+      privileged: true
+      config:
+        platform: linux
+        image_resource:
+          type: registry-image
+          source:
+            repository: vito/oci-build-task
+        inputs:
+          - name: docker-mongodb-exporter-git
+            path: .
+        outputs:
+          - name: image
+        caches:
+          - path: cache
+        run:
+          path: build
    - put: mongodb-exporter-image
      params:
-        build: docker-mongodb-exporter-git
-        tag_as_latest: true #(tag_as_latest is default in the new registry-image resource)
+        image: image/image.tar
```

## Adding extra auth credentials for registry-image build

### DockerHub

Example to add DockerHub credentals, eg. when being rate-limited for pulls:

```yaml
jobs:
  plan:
    - get: docker-mongodb-exporter-git
      trigger: true
    - task: create_creds
      config:
        platform: linux
        image_resource:
          type: registry-image
          source:
            repository: alpine
            tag: latest
            username: ((DOCKERHUB_USERNAME))
            password: ((DOCKERHUB_PASSWORD))
        outputs:
          - name: docker_creds
        run:
          path: sh
          args:
            - -exc
            - |
              AUTH="$(echo -n '((DOCKERHUB_USERNAME)):((DOCKERHUB_PASSWORD))' | base64 -w 0)"
              mkdir -p docker_creds
              cat > docker_creds/config.json <<EOF
              { "auths": { "https://index.docker.io/v1/": { "auth": "$AUTH" }}}
              EOF
    - task: docker-build
      privileged: true
      config:
        platform: linux
        image_resource:
          type: registry-image
          source:
            repository: vito/oci-build-task
        params:
          DOCKERFILE: docker/Dockerfile
          DOCKER_CONFIG: docker_creds
        inputs:
          - name: docker-mongodb-exporter-git
            path: .
          - name: docker_creds
        outputs:
          - name: image
        caches:
          - path: cache
        run:
          path: build
```

### ECR

Example to add login with aws to authenticate to ECR:

```yaml
jobs:
  plan:
    - get: docker-mongodb-exporter-git
      trigger: true
    - task: create_creds
      config:
        platform: linux
        image_resource:
          type: registry-image
          source:
            repository: skyscrapers/concourse-ecr-login
            tag: latest
        outputs:
          - name: docker_creds
        run:
          path: sh
          args:
            - -exc
            - |
              export AWS_ACCESS_KEY_ID=((OPS_AWS_ACCESS_KEY_ID))
              export AWS_SECRET_ACCESS_KEY=((OPS_AWS_SECRET_ACCESS_KEY))
              aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin ((OPS_AWS_ACCOUNT_ID)).dkr.ecr.((OPS_AWS_REGION)).amazonaws.com
              cp /root/.docker/config.json docker_creds/config.json
    - task: docker-build
      privileged: true
      config:
        platform: linux
        image_resource:
          type: registry-image
          source:
            repository: vito/oci-build-task
        params:
          DOCKERFILE: docker/Dockerfile
          DOCKER_CONFIG: docker_creds
        inputs:
          - name: docker-mongodb-exporter-git
            path: .
          - name: docker_creds
        outputs:
          - name: image
        caches:
          - path: cache
        run:
          path: build
```

## helm v3

At the time of writing the [concourse-helm-resource](https://github.com/linkyard/concourse-helm-resource/issues/135) is not compatible with Helm v3.
There is however a [concourse-helm3-resource](https://github.com/Typositoire/concourse-helm3-resource) forked from that repository that you can use. We are using that resource and are contributing actively to it.

Example usage:

```yaml
resource_types:
  - name: helm
    type: docker-image
    source:
      repository: typositoire/concourse-helm3-resource
resources:
  - name: example-helm
    type: helm
    source:
      cluster_url: https://example.eks.amazonaws.com/
      cluster_ca: ((HELM_CI.cluster_ca))
      token: ((HELM_CI.token))
      helm_history_max: 10
jobs:
  - name: example-deploy
    plan:
      - get: git-example-chart
        trigger: true
      - get: docker-example
        params:
          skip_download: true
        trigger: true
      - put: example-helm
        params:
          chart: git-example-chart/charts/example/
          values: git-example-chart/charts/example/values.yaml
          namespace: production
          release: example
          override_values:
            - key: image.sha256
              path: docker-example/digest
```

## Feature environments

Check out the [dedicated page on Feature Environments](./feature_environments.md) for an example on how you can implement this with Concourse.
