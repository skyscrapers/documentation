# Concourse

At Skyscrapers we've standardized on Concourse for our CI/CD system, both for ourselves and for our customers. It is an integral part of our latest Kubernetes & AWS ECS Reference Solutions

Concourse is a next generation continuous integration and delivery tool. Similar to configuration management tools, Concourse CI/CD pipelines are described as code and put under source control. This is a major improvement over other CI systems and setups that are often difficult to reproduce in case of failure.

You'll find your Concourse address in the `README.md` file of your GitHub repo. If you don't already have a Concourse setup, ping us and we can set it up for you.

## CLI

Concourse can be accessed either via the web UI or the CLI (`fly`). The web UI is very useful for getting a visual state of the pipelines, as well as triggering and cancelling jobs, but Concourse is primarily driven from the command-line. You can read more about the `fly` CLI in the [official Concourse documentation](https://concourse-ci.org/fly.html).

A bit further in this page you'll also find some highlights and examples of common commands about `fly`.

## Authentication

Authentication in Concourse is done via GitHub. You need to belong to a certain GitHub team to be able to access your customer team in Concourse. If you're able to access this documentation repo, chances are that you also have access to Concourse. If not, ping `@help` in Slack and we'll make sure you're added to the correct team.

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

You'll sometimes want to set secrets and other sensitive data in your pipelines, like AWS or DockerHub credentials. There are a couple of options to do that, if you have a Vault setup you can store your sensitive data in Vault and use it in Concourse via the [credential management support](https://concourse-ci.org/creds.html), or you can use pipeline parameters to keep your sensitive data out from the pipeline definitions.

### Vault - Concourse integration

The Vault integration solution is preferred, as there are less moving parts, secrets are more secure than a plain `yaml` file, and it's a much more robust system overall. But of course it can only be used if you already have a running Vault setup.

To learn how to use Vault secrets in your Concourse pipelines you should first head to the Concourse [Credential Management documentation](https://concourse-ci.org/creds.html).

Normally, Concourse should be configured to look for secrets at the path `concourse/<your-concourse-team-name>`, where `<your-concourse-team-name>` is the Concourse team name where the pipeline accessing the secret is running.

#### Examples

For example, if we want to set a secret token in one of our pipelines, we'll first write it in Vault like this:

```bash
vault write concourse/yourteamname/secrettoken value="supersecretconfidentiallongtoken"
```

Then you can set it in your pipeline definition with `((secrettoken))`. After you save the pipeline in Concourse, it will retrieve it from Vault every time it needs it. So if you update the secret in Vault, Concourse will automatically retrieve the new value.

You can also access specific values inside a secret. For example, you could write a secret that has both a username and a password, like this:

```bash
vault write concourse/yourteamname/useraccess username="john" password="123456789"
```

You could then access it in your pipeline with `((useraccess.username))` and `((useraccess.password))`.

**Caution, with Vault, if there are multiple values in a secret, e.g. username=x and password=y, all values need to be specified if you make a change.
As Vault overwrites the secret entry instead of updating a specific value. So if you update a password value, you need to include the username as well**

#### Limitations

At the moment, the Vault integration in Concourse is a bit limited (more info can be found in [this GitHub issue](https://github.com/concourse/concourse/issues/1814#issuecomment-352832086)), so the only secrets engine that we can use is the [KV secrets engine](https://www.vaultproject.io/docs/secrets/kv/index.html).

This integration is still useful to store static secrets like passwords, tokens or Git keys, so we don't have to provide them as plaintext via a `secrets.yaml` file, which is also quite hard to distribute to the team. But you have to be aware that for the moment this integration won't allow you to dynamically provision AWS credentials for example.

You can find more detailed information in the [Vault specific documentation](./vault.md).

### Plain secrets.yaml file

Another option for using sensitive data and information in your Concourse pipelines is to use [pipeline `((params))`](https://concourse-ci.org/setting-pipelines.html#pipeline-params). The syntax of the pipeline definition `yaml` is the same as the one for the Vault integration, with the difference that you'll need to provide the values of those parameters when setting the pipeline in Concourse with `fly -t youralias set-pipeline -p pipeline_name -c pipeline_definition.yaml -l secrets.yaml`. This way secrets are kept separate from your pipeline definitions and aren't committed to source control.

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

There's one important thing to consider here though: wheather you're using Vault for providing the pipeline `((params))` or not. If you're using Vault, then you don't need to make further changes in your current pipelines, as they'll keep picking up their `((params))` from Vault. But if you're using an external `params.yaml` or `secrets.yaml` file you might have some extra work ahead. If this is your case, keep reading.

### Pipeline automation without Vault

If you are **not** using Vault for your pipeline `((params))`, you could provide them via one of the following methods.

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
