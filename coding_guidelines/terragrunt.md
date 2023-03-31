# Terragrunt

- [Terragrunt](#terragrunt)
  - [Folder structure](#folder-structure)
    - [Standards we use](#standards-we-use)
  - [Terragrunt configuration](#terragrunt-configuration)
    - [Dependencies](#dependencies)
  - [Formatting](#formatting)
  - [Naming](#naming)
  - [AWS authentication](#aws-authentication)
  - [Usage](#usage)
  - [Install](#install)
  - [Secrets](#secrets)
    - [SOPS](#sops)
  - [Modules](#modules)
    - [Documentation](#documentation)

## Folder structure

Terragrunt projects should be organized using the following structure:

```console
<repository root>
└── terraform
  └── live                 # Contains the live representation of the infrastructure
      └── default          # Generic and cross-environment stacks
        └── bootstrap      # Base stack added via template
          └── (terraform files)
          └── terragrunt.hcl       # Terragrunt configuration
        └── sso            # AWS IAM/SSO configuration
          └── (terraform files)
          └── terragrunt.hcl       # Terragrunt configuration
      └── production
        └── my-first-application   # Application / workload stack
          └── terragrunt.hcl       # Terragrunt configuration
        └── my-other-application   # Application / workload stack
          └── terragrunt.hcl       # Terragrunt configuration
        └── eks
          └── (cluster-name)
            └── addons
              └── terragrunt.hcl       # Terragrunt configuration
            └── cluster
              └── terragrunt.hcl       # Terragrunt configuration
            └── custom-addons
              └── terragrunt.hcl       # Terragrunt configuration
        └── networking
          └── base
            └── terragrunt.hcl       # Terragrunt configuration
          └── peering
            └── terragrunt.hcl       # Terragrunt configuration
        └── route53
          └── (terraform files)
          └── terragrunt.hcl       # Terragrunt configuration
        └── env.hcl        # Contains environment-specific Terragrunt config
      └── ...
        └── ...
      └── .gitignore
      └── aws_provider.hcl # Terragrunt configuration that defines the common AWS Terraform provider
      └── eks_provider.hcl # Terragrunt configuration that defines the common EKS Terraform provider
      └── terragrunt.hcl   # Main Terragrunt configuration
  └── modules
      └── my-first-application   # Application / workload module
        └── (terraform files)
      └── my-other-application   # Application / workload module
        └── (terraform files)
      └── networking-peering     # VPC peering module
        └── (terraform files)
```

### Standards we use

- If code is re-used across multiple environments it should be made into a module. See also [modules](#modules)
- Variables that can be loaded in through a dependency should not be hardcoded. Instead they should be loaded in through a [dependency](#dependencies)

## Terragrunt configuration

Terragrunt configuration is defined in a terragrunt.hcl file. This uses the same HCL syntax as Terraform itself.

Example for a terragrunt file:

```hcl
include "root" {
  path = find_in_parent_folders()
}

include "env" {
  path = find_in_parent_folders("env.hcl")
}

include "aws_provider" {
  path = find_in_parent_folders("aws_provider.hcl")
}

terraform {
  source = "${get_path_to_repo_root()}/terraform/modules/rds"
}

dependency "networking_base" {
  config_path = "../networking/base"
}

inputs = {
  vpc_id                  = dependency.networking_base.outputs.vpc_id
  db_subnet_ids           = dependency.networking_base.outputs.private_db_subnets

  engine_mode    = "provisioned"
  engine_version = "5.7.mysql_aurora.2.11.1"
  instance_class = "db.r5.xlarge"
  instances      = { 1 = {} }
}
```

### Dependencies

In the example above you saw this part:

```hcl
dependency "networking_base" {
  config_path = "../networking/base"
}
```

If you want to make use of outputs from different environment stacks you can load them in via such dependencies. This way you can then later use them in the inputs section like this:

```hcl
vpc_id = dependency.networking_base.outputs.vpc_id
```

## Formatting

Terraform code should always be formatted by `terraform fmt -recursive ./`. This command will take care of all indentation, alignment, ...

**Note**: having the Terraform plugin installed also helps with this.

Variables and outputs should have a *clear description* what it is for and the expected format. For example:

```hcl
variable "client_sg_ids" {
  description = "Security group IDs for client access to the Structr instance(s)"
  type        = list(string)
  default     = null
}
```

```hcl
output "elb_dns_name" {
  description = "ELB DNS name for the frontend"
  value       = module.elb.elb_dns_name
}
```

## Naming

Resources, variables and outputs should *use `_` as a separator*.

Other than the general naming guidelines, Terraform **resource names** should:

- be truncated automatically if they are longer than the maximum allowed length
- **note* be suffixed with the type (eg. `"aws_iam_role" "billing"` vs `"aws_iam_role" "billing_role"`) as this is redundant already with the resource type. This also let's you keep names shorter, making it less likely to hit the character limit

And Terraform **variables and outputs** should:

- end with the type they're referring to, for example if the output is an instance ID, its name should be `vault_instance_id`, not `vault_instance`. This makes it much more clear what the actual output is.
- be singular if they're a single string or number, and plural if they're a list. For example, if an output contains a list of instance IDs, its name should be `vault_instance_ids`.

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
3. Assumed role | | 2. Assume         | Terraform state S3
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

- access to the S3 bucket containing the Terraform state files
- access to the DynamoDB table containing the Terraform state locks
- permission to assume a more privileged role in the target "ops" accounts

This is the [Hashicorp's recommended approach for multi-account AWS architectures](https://www.terraform.io/docs/backends/types/s3.html#multi-account-aws-architecture), and these are some of its benefits:

- we don't have to manage and secure static credentials with direct admin access to each "ops" accounts.
- the provided credentials from the assumed role last for just an hour, so it's more difficult that they get compromised.

## Usage

Terragrunt is a layer on top of Terraform. Therefore all the commands that can be used in Terraform can also be added to the terragrunt command.
More info: <https://terragrunt.gruntwork.io/docs/getting-started/quick-start/>

## Install

You can install Terragrunt through the package manager that you use.
More info: <https://terragrunt.gruntwork.io/docs/getting-started/install/>

## Secrets

All secrets such as passwords, certificates, ... must be encrypted.
You can do this using:

- KMS, see the [official docs how](https://www.terraform.io/docs/providers/aws/d/kms_secrets.html).
- SOPS with KMS backend, See the [official docs how](https://github.com/mozilla/sops)

You can re-use the KMS key used for Terraform encryption documented in the customer's documentation (`docs/terraform.md`. Usually this key is created through Terraform in the `general` stack.

### SOPS

In order to work with SOPS in terraform stacks the following components are needed:
A `.sops.yaml` file in corresponding environment folder (eg. `terraform/live/production/.sops.yaml` ) with the following config:

```yaml
---
creation_rules:
  - kms: "arn:aws:kms:eu-west-1:123456789012:key/11111111-111-1111-1111-111111111111"
    role: arn:aws:iam::123456789012:role/ops/admin
    aws_profile: <customer>SharedTooling
```

The following lines need to be added to the `terragrunt.hcl` file where you want to load in these secrets:

```hcl
locals {
  secret_vars = yamldecode(sops_decrypt_file("./secrets.yaml"))
}

inputs = merge({
  ...
}, local.secret_vars)
```

To create the sops secret file you can just run `sops secrets.yaml` in the folder where you want the secret to be saved. The content of this file is best structured as yaml.

## Modules

If you can re-use a set of Terraform code, consider adding it as a module.

We have a lot of general modules we can reuse for different clients. You can find them all on GitHub: <https://github.com/skyscrapers?utf8=%E2%9C%93&q=terraform-&type=&language=hcl>.
Next to our own modules there is also a large set of modules available in the Terraform community: <https://registry.terraform.io/browse/modules>.

Each module must have a `README.md` consisting of:

- A description of what it does.
- Which requirements does the module need.
- Configuration parameter documentation ([autogenerated](#documentation)).

### Documentation

You should use [terraform-docs](https://github.com/segmentio/terraform-docs) to automatically generate a variable table from terraform variables for use in documentation.

Use the following parameters:

```bash
terraform-docs markdown --sort-by required --escape=false <folder>
```

You can easily create a function for this which also copies the output to your clipboard. For example

```bash
tf-docs () { terraform-docs markdown --sort-by required --escape=false $1 | <your OS's clipboard> }
```
