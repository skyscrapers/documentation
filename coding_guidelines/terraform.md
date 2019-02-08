# Terraform

## Formatting

Terraform code should always be formatted by `terraform fmt`. This command will take care of all indentation, alignment, ...

Variables and outputs should have a clear description what it is for and the expected format. For example:

```hcl
variable "client_sg_ids" {
  description = "List(optional, []): Security group IDs for client access to the Structr instance(s)"
  type        = "list"
  default     = []
}
```

```hcl
output "elb_dns_name" {
  value       = "${module.elb.elb_dns_name}"
  description = "String, ELB DNS name for the frontend.
}
```

## Naming

Resources, variables and outputs should use `_` as a separator.
Other than the general naming guidelines, terraform names should:

* be truncated automatically if they are longer than the maximum allowed length
* end with the type they're referring to, for example if the output is an instance id, its name should be vault_instance_id, not vault_instance. This makes much more clear what the output is.
* be singular if they're a single string or number, and plural if they're a list. For example, if an output contains a list of instance ids, its name should be vault_instance_ids.

## Behaviour

All terraform code should work on the first apply. Applying the same code twice should not result in changes.

Variable values for different workspaces should be in separate `.tfvars` files, where the name should be the workspace name they're applied to. For example, a stack with two workspaces, staging and production, should also contain two tfvars files: `staging.tfvars` and `production.tfvars`. A stack with a `default.tfvars` file or without any `tfvars` files means that it only works with the `default` namespace.

## Folder structure

Terraform configuration should be organized using the following structure:

```console
<repository root>
└── terraform
    ├── modules
    └── stacks
        └── bootstrap
        └── general
        └── concourse
        └── ...
```

All folders in `<repository root>/terraform/stacks` should contain applyiable Terraform stacks or variable files for standard stacks.
The `<repository root>/terraform/modules` contain reusable modules specific to the repository they are in.

## Remote State

All terraform state has to be stored in an encrypted S3 bucket in the customer's "admin" account. The creation of this bucket, along with the needed IAM users and roles to access the AWS infrastructure, is handled by the `bootstrap` stack.

`stacks/bootstrap` is the first stack that should be run on any new infrastructure, and it's a special stack as it uses a local state, committed into source control. This is because before applying the `bootstrap` stack there is no S3 bucket to push the state to.

All the other Terraform stacks should contain a `terraform` block to configure the remote state, for example:

```hcl
terraform {
  required_version = ">= 0.11.11"

  backend "s3" {
    bucket         = "terraform-remote-state-example"
    key            = "stacks/concourse"
    region         = "eu-west-1"
    dynamodb_table = "terraform-remote-state-lock-example"
    encrypt        = true
    acl            = "bucket-owner-full-control"
    profile        = "ExampleAdmin"
  }
}
```

The `key` path in S3 should be the path of the stack relative to the `terraform` directory. For instance, the previous example refers to the following stack:

```console
terraform
└── stacks
    └── concourse
```

## AWS authentication

To authenticate Terraform to AWS, we use a delegated access approach. Instead of accessing direclty an "ops" account with some set of credentials, we authenticate with an "admin" account and configure the Terraform AWS provider to assume an admin role in the target "ops" account. See the diagram below.

```ascii
          1. User with
             access to
             admin account
                  +
                  |
                  |
                  v
            +-----+-----+
            |           |
            | Terraform +-------------+
            |           |             |
            +---+-+-----+             |
                | |                   |
                | |                   |
                | |                   | Direct access to the
3. assumed role | | 2. Assume         | Terraform state S3
   with temp.   | |    role in        | bucket and DynamoDB table
   credentials  | |    ops staging    |
      +---------+ |                   |
      |           v                   |
      |     +-----+------+            |
      |     |            |            |
      |     |   Admin    +<-----------+
      |     |   account  |
      |     |            |
      |     +------------+
      |
      v
+-----+------+          +------------+
|            |          |            |
|  Ops       |          | Ops        |
|  staging   |          | production |
|  account   |          | account    |
|            |          |            |
+------------+          +------------+
```

Each customer has an "admin" account and at least one "ops" account. The "admin" account is where the Terraform state is stored and where all the IAM users that need access to the infrastructure are created. The "ops" accounts are the ones containing the actual operational resources, like EC2 instances, load balancers, etc. Ideally, the "ops" accounts don't have IAM users with direct access, instead there are multiple IAM roles with different set of capabilities, which can be assumed by users from the "admin" account.

Following a least privilege approach the user running Terraform should have a set of credentials configured to access the "admin" account, with just the following permissions:

* access to the S3 bucket containing the Terraform state files
* access to the DynamoDB table containing the Terraform state locks
* permission to assume a more privileged role in the target "ops" accounts

The [`terraform-state` module](https://github.com/skyscrapers/terraform-state#output) already creates a IAM policy that has the necessary access rights to the S3 bucket and DynamoDB table. And, as explained in the [Remote state section](#remote-state), the permission to assume roles in the "ops" accounts is handled in the `bootstrap` stack.

This is the [Hashicorp's recommended approach for multi-account AWS architectures](https://www.terraform.io/docs/backends/types/s3.html#multi-account-aws-architecture), and these are some of its benefits:

* we don't have to manage and secure static credentials with direct admin access to each "ops" accounts.
* the provided credentials from the assumed role last for just an hour, so it's more difficult that they get compromised.
* if there are multiple "ops" accounts (like in the example above), we can still have the Terraform remote state centralized in one place, so we avoid having to share the S3 bucket accross accounts and having potential state ownership problems.

Normally, the Terraform AWS provider should be configured like this:

```hcl
provider "aws" {
  region              = "eu-west-1"
  profile             = "ExampleAdmin"
  allowed_account_ids = ["1234567890"]

  assume_role = {
    role_arn = "arn:aws:iam::1234567890:role/ops/admin"
  }
}
```

*Note that this is just an example to show how Terraform authenticates to AWS, but you'll normally put the `role_arn` and `allowed_account_ids` in variables so they can be set differently depending on which "ops" account you're targetting.*

## Secrets

All secrets such as passwords, certificates, ... must be encrypted.
You can do this using KMS, see the [official docs how](https://www.terraform.io/docs/providers/aws/d/kms_secrets.html).

## Modules

If you can re-use a set of Terraform code, consider adding it as a module.
We have a lot of general modules we can reuse for different clients. You can find them all in GitHub: <https://github.com/skyscrapers?utf8=%E2%9C%93&q=terraform-&type=&language=hcl>
Modules can be created for a specific customer, altough this is uncommon.

Each module must have a README.md consisting of:

* A description of what it does
* Which requirements does the module need
* Configuration parameter documentation

## Stacks

A Stack can refer to a deployable unit, or a standard stack, described below.
A Deployable unit is a set of resources containing everything needed to setup a service, or a sub-stack of that service when it makes sense to separate the `terraform apply` runs.

For example, the `k8s-cluster` stack has a `base` and `cluster` sub-stack. Both sub-stacks do different things, and each lifecycle needs to be controlled independently.

```console
stacks
├── concourse
│   ├── backend_config.tfvars
│   └── tools.tfvars
└── k8s-cluster
    ├── base
    │   ├── main.tf
    │   ├── outputs.tf
    │   ├── production.tfvars
    │   ├── staging.tfvars
    │   └── variables.tf
    ├── cluster
    │   ├── kops-cluster.yaml
    │   ├── main.tf
    │   ├── outputs.tf
    │   ├── production.tfvars
    │   ├── staging.tfvars
    │   └── variables.tf
    └── k8s-routing-tools
        ├── main.tf
        └── variables.tf
```

### Standard Stack

Building up on the Terraform modules concept, we also have standard stacks. A Terraform standard stack is a complete stack that you can deploy providing the needed variables and an S3 backend configuration. The goal of these stacks is to reduce code duplication and drift between different setups and customers.

You can find more information on the standard stacks in the initial proposal: <https://docs.google.com/document/d/1FmT21rjMoJWLiRpL4h-chaviPu-Lc2Cccb-bEAgoE_4/edit>

#### Customer folder structure

Although it's not required for this to work, as you can initialize Terraform wherever you want, it's important that we keep a homogenic folder structure for all of our setups, so it's more manageable and maintainable.

Considering this, every customer repository should contain a `terraform/stacks` folder, containing all the stacks deployed for that customer. Although, when using a standard stack like `teleport-server` or `concourse`, the stack source code won't reside in those folders in the customer repository, only the S3 backend configuration and the workspace variables will be there.

#### Usage

Normally all our standard Terraform stacks will follow the same usage patterns, which are documented below.

You'll first need to initialize Terraform with the backend configuration specific to the customer you are deploying to. You can provide that configuration via an HCL file (like a `tfvars` file) or via key/value assignments as command line flags. See the [Terraform documentation on partial configuration](https://www.terraform.io/docs/backends/config.html#partial-configuration) for more information. In this example we'll use a `tfvars` file provided in the customer repo.

**IMPORTANT**: Remember that once Terraform is initialized in a directory, it'll be configured to use the backend configuration you provided until it's reinitialized. With that in mind, **it's not recommended** to initialize Terraform in the stack repository folder, as it might cause some conflicts when trying to deploy that stack to multiple customers (if you forget to reinitialize).

To avoid possible conflicts and confusion, it's recommended to initialize Terraform in a customer-specific directory, and point each command to the stack source code path. In here, we'll use the [`teleport-server` stack](https://github.com/skyscrapers/teleport-server-stack) as an example, but it can be used for any other stack.

```console
cd customer/terraform/stacks/teleport-server/
terraform init -backend-config backend_config.tfvars ../../../path/to/the/teleport-server/stack
```

*See Tips & tricks below, to not have to use the path everytime.*

Once initialized, select or create the appropriate workspace.

```console
terraform workspace select tools ../../../path/to/the/teleport-server/stack
```

Then you can plan and apply as you would normally do.

```console
terraform apply -var-file tools.tfvars ../../../path/to/the/teleport-server/stack
```

**Note** that you'll need to point to the Terraform stack path in all commands.

#### Caveats and future improvements

##### Stack versioning

By using the Terraform standard stacks as mentioned before, it'll be hard to track which version of the stack we've deployed for a customer, and we'll need to set the standard stack local repository to the correct version every time we need to deploy it. A possible solution to this would be to use git submodules in the customer repositories to point to the stack's repositories, that way we can lock a specific version for each setup.

## Automated testing

It is a good practice to write tests to ensure that your code does what it is expected to do, in a repeatable and predictable way, and Terraform is no exception to that rule. After doing some research, we decided to go with Terratest for our automated tests for Terraform. You can find an example in our terraform-vault module. Also, these tests should, ideally, run automatically in a CI. In our case we have a pipeline in Concourse for all our Terraform modules: <https://ci.skyscrape.rs/teams/skyscrapers/pipelines/terraform-modules/>. So if you add tests to a Terraform module, make sure to add them to that pipeline.

**Important**
Note that Terraform tests create real resources in AWS, so make sure your tests also run a clean-up step to destroy everything they create, so there are no left-overs that could cost us money. To mitigate this, tests should run on an isolated AWS test account, where we could potentially wipe out everything at any time. There´s a couple of issues for this: <https://github.com/skyscrapers/engineering/issues/37> and <https://github.com/skyscrapers/engineering/issues/38>

## Tips & tricks

### Standard stacks

Having to input the stack path on every terraform command can be a hassle, to improve the usage a bit we can use a bash function like this:

Define the following function in your `.bash_profile` or `.zshrc`:

```console
tf() { terraform "$@" $TF_STACK_PATH;}
```

Then, when you're working on a specific Terraform stack, you just need to point the `$TF_STACK_PATH` environment variable to the stack's absolute path and invoke terraform with `tf`. For example:

```console
export TF_STACK_PATH=../../../path/to/the/teleport-server/stack
tf init -backend-config backend_config.tfvars
tf workspace select tools
tf apply -var-file tools.tfvars
```

### Documentation

You can use [terraform-docs](https://github.com/segmentio/terraform-docs) to automatically generate a variable table from terraform variables for use in documentation.
