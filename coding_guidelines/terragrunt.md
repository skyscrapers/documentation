# Terragrunt

- [Terragrunt](#terragrunt)
  - [Folder structure](#folder-structure)
  - [Terragrunt configuration](#terragrunt-configuration)
  - [Formatting](#formatting)
  - [Naming](#naming)
  - [AWS authentication](#aws-authentication)
  - [Usage](#usage)
  - [Install](#install)
  - [Secrets](#secrets)
  - [Modules](#modules)
    - [Standard Stack](#standard-stack)
    - [Documentation](#documentation)


## Folder structure

Terragrunt projects should be organized using the following structure:

```console
<repository root>
└── terraform
  ├── modules
  └── live                 # Contains the live representation of the infrastructure
      └── default          # Generic and cross-environment stacks
        └── bootstrap      # Base stack added via template
        └── general
        └── iam
          └── kube
          └── ops
        └── env.hcl        # Contains environment-specific Terragrunt config
      └── production
        └── example-stack
        └── networking
        └── rds
            └── project_a
            └── project_b
        └── env.hcl        # Contains environment-specific Terragrunt config
      └── ...
      └── aws_provider.hcl # Terragrunt configuration that defines the common AWS Terraform provider
      └── eks_provider.hcl # Terragrunt configuration that defines the common EKS Terraform provider
      └── terragrunt.hcl   # Main Terragrunt configuration
```

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

You can re-use the KMS key used for Terraform encryption documented in the customer's documentation (eg. `docs/terraform.md`, see [Customer Template](#customer-template)). Usually this key is created through Terraform in the `general` stack.

## Modules

If you can re-use a set of Terraform code, consider adding it as a module.

We have a lot of general modules we can reuse for different clients. You can find them all on GitHub: <https://github.com/skyscrapers?utf8=%E2%9C%93&q=terraform-&type=&language=hcl>.
Next to our own modules there is also a large set of modules available in the Terraform community: <https://registry.terraform.io/browse/modules>.

Modules can be created for a specific customer, altough this is uncommon. Usually when a customer-specific module gets created, it will get generalized later by our `#c_platform` circle to be able to use accros multiple customers.

Each module must have a `README.md` consisting of:

- A description of what it does
- Which requirements does the module need
- Configuration parameter documentation ([autogenerated](#Documentation))

### Standard Stack

Building up on the Terraform modules concept, we also have standard stacks. A Terraform standard stack is a complete stack that you can deploy providing the needed variables and an S3 backend configuration. The goal of these stacks is to reduce code duplication and drift between different setups and customers.
The Kubernetes and networking stacks are some of the examples.

### Documentation

You should use [terraform-docs](https://github.com/segmentio/terraform-docs) to automatically generate a variable table from terraform variables for use in documentation.

Use the following parameters:

```bash
terraform-docs --sort-by-required --no-escape markdown <folder>
```

Note: the `--no-escape` parameter is coming soon...

You can easily create a function for this which also copies the output to your clipboard. For example

```bash
tf-docs () { terraform-docs markdown --sort-by-required --no-escape $1 | <your OS's clipboard> }
```
