## Vault

We use Vault to store security information for our automated processes.

We have a HA vault configuration with a DynamoDB backend.

Both vault servers are configured with Teleport for SSH management with
2 route53 records providing access to the individual instances.

[Skyscrapers Terraform for setting up tools including Vault](https://github.com/skyscrapers/skyscrapers-terraform)

[Skyscrapers Vault Terraform moudule](https://github.com/skyscrapers/terraform-vault)

## Using Vault

The client Binary can be downloaded from here:

[HashiCorp Latest Version](https://www.vaultproject.io/downloads.html)
[HashiCorp Previous Versions](https://releases.hashicorp.com/vault/)

### To interact with Vault

Once you have authenticated to Vault for a customer, you will remain authenticated between sessions.

```bash
export VAULT_ADDR=https://vault.production.skyscrape.rs
```

This is only needs to be down once for each customer.

```bash
vault login -method=github
```

Get the status and view the configured secrets within a secret engine path.

```bash
vault status
vault list concourse/<customer-name>
```

**Currently it is not possible to list which secrets engines are enabled, e.f. KV is the key value pair engine.**

```bash
vault secrets list

Error listing secrets engines: Error making API request.

URL: GET https://vault.production.skyscrape.rs/v1/sys/mounts
Code: 403. Errors:

* permission denied
```

### Reading a secret

`vault read concourse/<customer-name>/name-of-secret`

See [concourse](./concourse.md) for how these can be used.

### Writing a value to Vault

The format of a secret is key-value with the name being the key.

`vault write concourse/<customer-name>/your-secret value=shhhhhh-this-is-a-secret`

It is possible to have multiple values within a secret:

`vault write concourse/<customer-name>/your-secret value1=this-is-the-first-secret value2=this-is-the-second-secret`

**Caution - if a secret has multiple values, they all have to be entered/updated each time, as Vault overrites, it does not update .**

As an example, we set a username and password for a secret.
Then we want to update the password, but if the username is not included, the secret will only have the password.

1. `vault write concourse/<customer-name>/some-creds username=somebody password="not-telling"`

```bash
 vault read concourse/<customer-name>/some-creds

 Key                 Value
 ---                 -----
 refresh_interval    768h
 password            not-telling
 username            somebody

```

2. `vault write concourse/<customer-name>/some-creds password="itsAsecret"`

```bash
vault read concourse/<customer-name>/some-creds

Key                 Value
---                 -----
refresh_interval    768h
password            itsAsecret
```

### Removing a secret

```bash
vault delete concourse/<customer-name>/some-creds

Success! Data deleted (if it existed) at: concourse/<customer-name>/some-creds
```
