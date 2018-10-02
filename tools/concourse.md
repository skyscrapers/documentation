# Concourse

At Skyscrapers we've standardized on Concourse for our CI/CD system, both for ourselves and for our customers. It is an integral part of our latest Kubernetes & AWS ECS Reference Solutions

Concourse is a next generation continuous integration and delivery tool. Similar to configuration management tools, Concourse CI/CD pipelines are described as code and put under source control. This is a major improvement over other CI systems and setups that are often difficult to reproduce in case of failure.

You'll find your Concourse address in the `README.md` file of your GitHub repo. If you don't already have a Concourse setup, ping us and we can set it up for you.

## CLI

Concourse can be accessed either via the web UI or the CLI (`fly`). The web UI is very useful for getting a visual state of the pipelines, as well as triggering and cancelling jobs, but Concourse is primarily driven from the command-line. You can read more about the `fly` CLI in the official Concourse documentation: https://concourse-ci.org/fly.html

A bit further in this page you'll also find some highlights and examples of common commands about `fly`.

## Authentication

Authentication in Concourse is done via GitHub. You need to belong to a certain GitHub team to be able to access your customer team in Concourse. If you're able to access this documentation repo, chances are that you also have access to Concourse. If not, ping `@help` in Slack and we'll make sure you're added to the correct team.

### Logging in with fly

`fly` can be used to access multiple Concourse servers, to accomplish that it has the concept of targets. Each target represents a different Concourse setup, and you need to provide the alias of such target to all the `fly` commands with the `-t` flag, so it knows which Concourse server to reach. You can choose the name of your targets, it can be an arbitrary string, and it'll only apply to your computer.

```shell
fly -t youralias login --team-name yourcompanyname --concourse-url https://ci.yourcompanyname.com
```

*Note*: from here on you'll need to specify the `youralias` target in each `fly` command with `-t youralias`.

## How to set pipelines

```shell
fly -t youralias set-pipeline -p <pipeline-name> -c <path-to-the-ci-dir>/pipelines/<pipeline-name>.yaml -l secrets.yaml
```

For example:

```shell
fly -t youralias set-pipeline -p k8s-staging -c ci/pipelines/k8s-staging.yaml -l secrets.yaml
```

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

Another option for using sensitive data and information in your Concourse pipelines is to use [pipeline `((params))`](https://concourse-ci.org/setting-pipelines.html#pipeline-params). The syntax of the pipeline definition `yaml` is the same as the one for the Vault integration, with the difference that you'll need to provide the values of those parameters when setting the pipeline in Concourse with `fly set-pipeline`. This way secrets are kept separate from your pipeline definitions and aren't committed to source control.

The downside of this approach is that if the values of those parameters change, you'll have to set the pipeline again with the updated data. Another inconvenience is that it's hard to distribute the parameters file (`secrets.yaml`) to the team, specially when it's frequently updated, as it's ignored by Git. We normally store it internally in our shared password manager, so if you need the latest `secrets.yaml` file you can ask someone from Skyscrapers to provide it (`@help`).
