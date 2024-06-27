# Vault

We run Vault on Kubernetes based on [Hashicorp's helm chart](https://github.com/hashicorp/vault-helm).

We have a HA Vault configuration with a DynamoDB as backend, that is able to [auto unseal](https://www.vaultproject.io/docs/concepts/seal#auto-unseal) via KMS (also see [Auto-unseal considerations](#auto-unseal-considerations)).

- [Vault](#vault)
  - [Initializing vault](#initializing-vault)
  - [Using Vault](#using-vault)
    - [Authentication](#authentication)
    - [Get the status of Vault](#get-the-status-of-vault)
    - [Reading a secret](#reading-a-secret)
    - [Writing a secret to the KV secrets engine](#writing-a-secret-to-the-kv-secrets-engine)
    - [Removing a secret](#removing-a-secret)
    - [Injecting Vault secrets in your Pods](#injecting-vault-secrets-in-your-pods)
    - [Best practices](#best-practices)
      - [Hide Vault commands from your bash history](#hide-vault-commands-from-your-bash-history)
  - [Auto-unseal considerations](#auto-unseal-considerations)

## Initializing vault

The setup of Vault is partially automated by our Concourse setup. However the customer is required to run the Vault initialization (so the recovery keys are owned by the customer and encrypted with the keybase accounts of the users).

*Note This should be done in sync with Skyscrapers during the initial deployment of the cluster:*

```bash
kubectl exec -it -n vault vault-0 -- vault operator init -tls-skip-verify -recovery-shares=5 -recovery-threshold=3 -recovery-pgp-keys=<keybase:user1,keybase:user2, ...> -root-token-pgp-key=<keybase:user>
```

After initialization the root token needs to be encrypted with KMS in the AWS account of the customer and stored encrypted in the cluster definition file (found in the `k8s-cluster` folder of your shared github repository). This is required to be able to run the further configuration of the Vault cluster (Auth backends, policies, monitoring,...).

> [!IMPORTANT]
> Although Vault calls them `Recovery keys` it is important to note that these **can not** decrypt the master key and thus are not sufficient to unseal Vault if the AutoUnseal mechanism isn't working. They are purely an authorization mechanism when a quorum of users is required, eg. for generating a root token. Also see [Auto-unseal considerations](#auto-unseal-considerations).

## Using Vault

The client binary can be downloaded from [here](https://www.vaultproject.io/downloads.html).

If you have it enabled, you can also use the web UI for most operations. You can access the web UI using the same address you use in the Vault cli.

> [!IMPORTANT]
> Note that Vault is only accessible from within the VPC so you need to be connected to the VPN of your cluster or setup a port-forward via `kubectl`. If not using the external URL (`https://vault.<environment>.eks.yourcompanyname.com:8200`) for connecting to Vault, you will need to specify `-tls-skip-verify` with your commands.

### Authentication

You'll first need to configure the Vault address you want to target.
You'll find your Vault address in the `README.md` file of your GitHub repo. If you don't already have a Vault setup, ping us and we can set it up for you.

```bash
export VAULT_ADDR=https://vault.<environment>.eks.yourcompanyname.com:8200
```

Then you need to authenticate to Vault. Same as with Concourse, you can use [GitHub authentication](https://www.vaultproject.io/docs/auth/github.html). The following command will ask you for a GitHub personal token, make sure the token you provide has the `read:org` scope.

```bash
vault login -method=github
```

*Note to [configure an authentication backend for your team (like GitHub)](https://www.vaultproject.io/docs/auth).*

*Note that the Skyscrapers team needs to login with `vault login -method=github -path=github-sky`.*

### Get the status of Vault

Get the status and view the configured secrets within a secret engine path.

```bash
vault status
vault list <secret backend>/
```

### Reading a secret

```bash
vault read <secret backend>/<path to your secret>
```

See the [Concourse specific documentation](../Concourse/README.md) for how Vault secrets can be used within Concourse.

### Writing a secret to the KV secrets engine

Each secret in a [KV secrets engine](https://www.vaultproject.io/docs/secrets/kv/index.html) consists of a list of key-value pairs:

`vault write concourse/<your-concourse-team-name>/your-secret value=shhhhhh-this-is-a-secret`

or

`vault write concourse/<your-concourse-team-name>/your-secret value1=this-is-the-first-secret value2=this-is-the-second-secret`

> [!CAUTION]
> If a secret has multiple key-value pairs, they all have to be defined in each `vault write` command, as Vault overwrites the whole secret.

As an example, we set a username and password for a secret. Then we want to update the password, but if the username is not included, the secret will only have the password.

1. `vault write concourse/<your-concourse-team-name>/some-creds username=somebody password="not-telling"`

    ```bash
    vault read concourse/<your-concourse-team-name>/some-creds

    Key                 Value
    ---                 -----
    refresh_interval    768h
    password            not-telling
    username            somebody
    ```

2. `vault write concourse/<your-concourse-team-name>/some-creds password="itsAsecret"`

    ```bash
    vault read concourse/<your-concourse-team-name>/some-creds

    Key                 Value
    ---                 -----
    refresh_interval    768h
    password            itsAsecret
    ```

### Removing a secret

```bash
vault delete concourse/<your-concourse-team-name>/some-creds

Success! Data deleted (if it existed) at: concourse/<your-concourse-team-name>/some-creds
```

### Injecting Vault secrets in your Pods

The Vault setup on our K8s clusters include the [Vault Agent Injector](https://www.vaultproject.io/docs/platform/k8s/injector/). This injector alters Pod specs on-the-fly, based on annotations, to launch a Vault-agent sidecar container. This agent takes care of authenticating to vault and allows the containers in your Pod to consume Vault secrets via a shared volume.

First make sure you have appropriate [policies](https://www.vaultproject.io/docs/concepts/policies/) and [K8s auth backend roles (step 3 of configuration)](https://www.vaultproject.io/docs/auth/kubernetes/#configuration) defined in Vault, for example:

```bash
cat <<EOF > policy.hcl
path "secret/*" {
  capabilities = ["list", "read"]
}
EOF

vault policy write demo policy.hcl
vault write auth/kubernetes/role/demo bound_service_account_names=demo bound_service_account_namespaces=default policies=demo ttl=1h
```

Then configure your Pod's ServiceAccount and add the Vault annotations, for example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: vault-demo
  namespace: default
  annotations:
    vault.hashicorp.com/role: "demo"
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/agent-inject-secret-<unique-name>: "/path/to/secret"
spec:
  containers:
    - name: alpine
      image: alpine:latest
      command:
        - "cat"
        - "/vault/secrets/<unique-name>"
  serviceAccountName: demo
```

Once your Pod launches, the Vault agent will init and expose your secret(s) via volume on path: `/vault/secrets/<unique-name>`.

For more info, please consult the Vault documentation on K8s integration:

- <https://www.vaultproject.io/docs/platform/k8s/injector/>
- <https://www.vaultproject.io/docs/auth/kubernetes/>

> [!IMPORTANT]
> A service account must be present to use the Vault Agent Injector. It is ****not recommended** to bind Vault roles to the default service account provided to pods if no service account is defined.

### Best practices

#### Hide Vault commands from your bash history

Vault commands usually contain sensitive data, specially write commands. To keep that information secure it's important to avoid leaking it into the shell history file.

[This Vault guide](https://learn.hashicorp.com/vault/secrets-management/sm-static-secrets.html#q-how-do-i-enter-my-secrets-without-exposing-the-secret-in-my-shell-39-s-history-) shows you three options to do this.

## Auto-unseal considerations

Due to the deployment method of running Vault on Kubernetes, meaning Pods can be replaced at any time (rolling upgrades, node failures, rebalancing, ...) we use Vault's [Auto Unseal mechanism](https://www.vaultproject.io/docs/concepts/seal#auto-unseal) so no human interaction is needed during such events. This feature delegates the responsibility of securing the unseal key from users to a trusted service, which in our case is [AWS KMS](https://aws.amazon.com/kms/). This works very well, **however also comes with a big caveat**.

> [!CAUTION]
> This Auto Unseal KMS key is the **only** key able to decrypt Vault's master key, which means if the KMS key becomes unavailable (deleted, AWS disaster, ...) it is no longer possible to unseal Vault and get to it's contents. As mentioned earlier in this document, at time of writing, the "recovery keys" generated during Vault initialization are not able to decrypt the master key either.

For the moment, if some form of recovery is required, we can make use of AWS' new [multi-Region keys](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html). This would protect against an AWS region becoming unavailble for example.

As there's quite some demand for this, we hope eventually Vault will include a native recovery mechanism. More information can be found at the following sources:

- [Unseal Vault with Recovery Key ( Lost AWS KMS used for Auto unseal )](https://github.com/hashicorp/vault/issues/11244)
- [Feature Request: Support multiple KMS Keys for Auto Unseal](https://github.com/hashicorp/vault/issues/6046)
- [Switching to different aws kms key id (with the same key material)](https://discuss.hashicorp.com/t/switching-to-different-aws-kms-key-id-with-the-same-key-material/19116/22)
