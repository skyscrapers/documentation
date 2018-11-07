# Vault

We use Vault to store security information for our automated processes.

We have a HA vault configuration with a DynamoDB backend.

Both vault servers are configured with Teleport for SSH management with
2 route53 records providing access to the individual instances.

[This is how Vault is currently being set up](https://github.com/skyscrapers/terraform-vault)

## Using Vault

The client binary can be downloaded from here: https://www.vaultproject.io/downloads.html

### Authentication

You'll first need to configure the Vault address you want to target.
You'll find your Vault address in the `README.md` file of your GitHub repo. If you don't already have a Vault setup, ping us and we can set it up for you.

```bash
export VAULT_ADDR=https://vault.tools.yourcompanyname.com
```

Then you need to authenticate to Vault. Same as with Concourse, you can use [GitHub authentication](https://www.vaultproject.io/docs/auth/github.html). The following command will ask you for a GitHub personal token, make sure the token you provide has the `read:org` scope.

```bash
vault login -method=github
```

*Note that the Skyscrapers team needs to login with `vault login -method=github -path=github-sky`.*

### Get the status of Vault

Get the status and view the configured secrets within a secret engine path.

```bash
vault status
vault list concourse/<your-concourse-team-name>
```

*Note that currently it is not possible to list which secrets engines are enabled, e.f. KV is the key value pair engine.*

```bash
vault secrets list

Error listing secrets engines: Error making API request.

URL: GET https://vault.tools.yourcompanyname.com/v1/sys/mounts
Code: 403. Errors:

* permission denied
```

### Reading a secret

`vault read concourse/<your-concourse-team-name>/name-of-secret`

See the [Concourse specific documentation](./concourse.md) for how these can be used.

### Writing a secret to the KV secrets engine

Each secret in a [KV secrets engine](https://www.vaultproject.io/docs/secrets/kv/index.html) consists of a list of key-value pairs:

`vault write concourse/<your-concourse-team-name>/your-secret value=shhhhhh-this-is-a-secret`

or

`vault write concourse/<your-concourse-team-name>/your-secret value1=this-is-the-first-secret value2=this-is-the-second-secret`

**Caution - if a secret has multiple key-value pairs, they all have to be defined in each `vault write` command, as Vault overwrites the whole secret.**

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
