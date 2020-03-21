# Terraform

- [Terraform](#terraform)
  - [Formatting](#formatting)
  - [Naming](#naming)
  - [Behaviour](#behaviour)
  - [Folder structure](#folder-structure)
  - [Remote State](#remote-state)
  - [AWS authentication](#aws-authentication)
  - [Secrets](#secrets)
  - [Modules](#modules)
  - [Stacks](#stacks)
    - [Standard Stack](#standard-stack)
      - [Customer folder structure](#customer-folder-structure)
      - [Usage](#usage)
  - [Automated testing](#automated-testing)
  - [Documentation](#documentation)
    - [Customer template](#customer-template)
- [Get list of nodes](#get-list-of-nodes)
- [Create SSH tunnel to the RDS database](#create-ssh-tunnel-to-the-rds-database)
  - [Tips & tricks](#tips--tricks)
    - [Standard stacks](#standard-stacks)

## Formatting

Terraform code should always be formatted by `terraform fmt`. This command will take care of all indentation, alignment, ...

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
- **not** be suffixed with the type (eg. `"aws_iam_role" "billing"` vs `"aws_iam_role" "billing_role"`) as this is redundant already with the resource type. This also let's you keep names shorter, making it less likely to hit the character limit

And Terraform **variables and outputs** should:

- end with the type they're referring to, for example if the output is an instance ID, its name should be `vault_instance_id`, not `vault_instance`. This makes it much more clear what the actual output is.
- be singular if they're a single string or number, and plural if they're a list. For example, if an output contains a list of instance IDs, its name should be `vault_instance_ids`.

## Behaviour

All Terraform code should work on the first apply. Applying the same code twice should not result in changes.

Variable values for different workspaces should be in separate `.tfvars` files, where the name should be the workspace name they're applied to. For example, a stack with two workspaces, staging and production, should also contain two tfvars files: `staging.tfvars` and `production.tfvars`. A stack with a `default.tfvars` file or without any `tfvars` files means that it only works with the `default` namespace.

**Note**: Even if there are no workspace specific vars, there should be an empty `.tfvars.` file defined for the workspace. This is not to confuse/break some of our automations, and also makes it immediately clear which workspaces are available.

## Folder structure

Terraform configuration should be organized using the following structure:

```console
<repository root>
└── terraform
    ├── modules
    └── stacks
        └── bootstrap
        └── elasticsearch
        └── concourse
        └── general
        └── iam
            └── kube2iam
            └── ops
        └── networking
        └── networking-vpc-peering
        └── r53
        └── rds
            └── project_a
            └── project_b
        └── teleport-server
        └── ...
```

All folders in `<repository root>/terraform/stacks` should contain applyiable Terraform stacks or variable files for standard stacks.
The `<repository root>/terraform/modules` contain reusable modules specific to the repository they are in.

It's preferred to split up Terraform stacks per logical resource (`stacks/rds/<project>`, `stacks/s3/<project>`, ...), instead of bundling a lot of things together (like we used to do in eg. `static`). This allows for better maintainability, smaller `apply` changes and thus errors, easier automation and easier to add specific IAM resources etc.

## Remote State

All Terraform state has to be stored in an encrypted S3 bucket in the customer's "admin" account. The creation of this bucket, along with the needed IAM users and roles to access the AWS infrastructure, is handled by the `bootstrap` stack.

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

The [`terraform-state` module](https://github.com/skyscrapers/terraform-state#output) already creates a IAM policy that has the necessary access rights to the S3 bucket and DynamoDB table. And, as explained in the [Remote state section](#remote-state), the permission to assume roles in the "ops" accounts is handled in the `bootstrap` stack.

This is the [Hashicorp's recommended approach for multi-account AWS architectures](https://www.terraform.io/docs/backends/types/s3.html#multi-account-aws-architecture), and these are some of its benefits:

- we don't have to manage and secure static credentials with direct admin access to each "ops" accounts.
- the provided credentials from the assumed role last for just an hour, so it's more difficult that they get compromised.
- if there are multiple "ops" accounts (like in the example above), we can still have the Terraform remote state centralized in one place, so we avoid having to share the S3 bucket accross accounts and having potential state ownership problems.

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

You should document the KMS key used for Terraform encryption in the customer's documentation (eg. `docs/terraform.md`, see [Customer Template](#customer-template)). Usually this key is created through Terraform in the `general` stack.

## Modules

If you can re-use a set of Terraform code, consider adding it as a module.

We have a lot of general modules we can reuse for different clients. You can find them all on GitHub: <https://github.com/skyscrapers?utf8=%E2%9C%93&q=terraform-&type=&language=hcl>

Modules can be created for a specific customer, altough this is uncommon. Usually when a customer-specific module gets created, it will get generalized later by our `#engineering` domain to be able to use accros multiple customers.

Each module must have a `README.md` consisting of:

- A description of what it does
- Which requirements does the module need
- Configuration parameter documentation ([autogenerated](#Documentation))

## Stacks

A Stack can refer to a deployable unit, or a standard stack, described below.

A Deployable unit is a set of resources containing everything needed to setup a service, or a sub-stack of that service when it makes sense to separate the `terraform apply` runs.

For example, the `rds` stack has a `project_a` and `project_b` sub-stack. Both sub-stacks do different things, and each lifecycle needs to be controlled independently.

```console
stacks
├── concourse
│   ├── backend_config.tfvars
│   └── tools.tfvars
└── rds
    ├── project_a
    │   ├── main.tf
    │   ├── outputs.tf
    │   ├── production.tfvars
    │   ├── staging.tfvars
    │   └── variables.tf
    └── project_b
        ├── main.tf
        ├── outputs.tf
        ├── production.tfvars
        ├── staging.tfvars
        └── variables.tf
```

### Standard Stack

Building up on the Terraform modules concept, we also have standard stacks. A Terraform standard stack is a complete stack that you can deploy providing the needed variables and an S3 backend configuration. The goal of these stacks is to reduce code duplication and drift between different setups and customers.

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

## Automated testing

It is a good practice to write tests to ensure that your code does what it is expected to do, in a repeatable and predictable way, and Terraform is no exception to that rule. After doing some research, we decided to go with Terratest for our automated tests for Terraform. You can find an example in our terraform-vault module. Also, these tests should, ideally, run automatically in a CI. In our case we have a pipeline in Concourse for all our Terraform modules: <https://ci.skyscrape.rs/teams/skyscrapers/pipelines/terraform-modules/>. So if you add tests to a Terraform module, make sure to add them to that pipeline.

**Important**
Note that Terraform tests create real resources in AWS, so make sure your tests also run a clean-up step to destroy everything they create, so there are no left-overs that could cost us money. To mitigate this, tests should run on an isolated AWS test account, where we could potentially wipe out everything at any time. There´s a couple of issues for this: <https://github.com/skyscrapers/engineering/issues/37> and <https://github.com/skyscrapers/engineering/issues/38>

## Documentation

You should use [terraform-docs](https://github.com/segmentio/terraform-docs) to automatically generate a variable table from terraform variables for use in documentation.

Use the following parameters:

```bash
terraform-docs --sort-by-required --no-escape markdown <folder>
```

Note: the `--no-escape` parameter is coming soon...

You can easily create a function for this which also copies the output to your clipboard. For example

```bash
tf-docs () { terraform-docs --sort-by-required --no-escape markdown $1 | <your OS's clipboard> }
```

### Customer template

``````md
# Terraform

These environments are deployed in the `<ops account ID + name>` AWS account, and the Terraform state is stored in the `<admin account ID + name>` account.

Code and structure follows [our Terraform guidelines](https://github.com/skyscrapers/documentation/blob/master/coding_guidelines/terraform.md).

## Organisation

*Use this section to describe structure etc. of the customer's Terraform project. For example:*

Accounts:

- `000000000000 CustomerAdmin`
- `000000000000 CustomerStaging`
- `000000000000 CustomerProduction`

Most of our Terraform code to configure this setup is available in this Git repository, under the `terraform`
folder. All resources are set up in a number of layers or `stacks`:

- `bootstrap`
  - Sets up the [remote state](#remote-state) S3 bucket, DynamoDB table, IAM roles etc and everything needed for billing
  - Needs to be applied as the very first stack
  - Terraform workspaces: `default`
- `ecr`
  - Contains the ECR repositories
  - Terraform workspaces: `default`
- `general`
  - Contains some global AWS account resources, like KMS keys
  - Terraform workspaces: `staging` and `production`
- `iam`
  - `general`
    - Contains the IAM users that are managed on the `CustomerAdmin` AWS account
    - Terraform workspaces: `default`
  - `kube2iam`
    - Contains the roles that the applications can assume through the K8s workers to have access to AWS components
    - Terraform workspaces: `staging` and `production`
  - `ops`
    - Contains the IAM roles that we can assume through our users on the `CustomerAdmin` AWS acount
    - Terraform workspaces: `staging` and `production`
- `networking`
  - Main networking setup: VPC, subnets, ... (uses the [networking-stack](https://github.com/skyscrapers/networking-stack))
  - Terraform workspaces: `staging` and `production`
- `networking-vpc-peering`
  - VPC peering setup: VPC peering and routes to MongoDB Atlas and TimeScale
  - Terraform workspaces: `staging` and `production`
- `teleport-server`
  - Sets up a Teleport server (uses the [teleport-server-stack](https://github.com/skyscrapers/teleport-server-stack))
  - Terraform workspaces: `tools`
- `rds`
  - `project`
    - Creates the mysql RDS servers
    - Terraform workspaces: `production`
- `mysql`
  - `project`
    - Creates the users and databases on the mysql RDS
    - Terraform workspaces: `production`
- `s3`
  - `project`
    - Creates the S3 buckets and IAM access that is needed for S3
    - Terraform workspaces: `dev`, `staging` and `production`

## Authentication

To run these stacks, you'll need to have the `<admin account name>` profile configured and with valid credentials in your local `awscli` configuration.

## Encryption

To encrypt variables you need to do the following:

```shell
echo -n 'value I want to encrypt' > /tmp/plaintext-password
aws kms encrypt --key-id <KMS key ID> --plaintext fileb:///tmp/plaintext-password --encryption-context my=context --output text --query CiphertextBlob
rm /tmp/plaintext-password
```

KMS key IDs:
- Staging: `00000000-0000-0000-0000-000000000000`
- Production: `00000000-0000-0000-0000-000000000000`

**Important**: Don't forget to change `--encryption-context my=context` to a key/value pair that gives context to your key you want to encrypt.

## MySQL

For applying the mysql stack(s), you need to first setup an SSH tunnel through Teleport to gain access to the VPC and RDS database. Use one of the EKS worker nodes as jumphost:

```bash
# Get list of nodes
tsh ls --cluster <customer Teleport server> project=<EKS cluster name>

# Create SSH tunnel to the RDS database
tsh ssh --cluster <customer Teleport server> -L 3306:<RDS endpoint>:3306 root@workers-<EKS cluster name>-<instance_id>
```

For example:

```bash
PLACE EXAMPLE HERE
```
``````

## Tips & tricks

### Standard stacks

Having to input the stack path on every Terraform command can be a hassle, to improve the usage a bit we can use a bash function like this:

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

**Note**: Within Skyscrapers, we provide a more extensive script wrapping around all Terraform commands: <https://github.com/skyscrapers/skyscrapers-tools#terraform-helper>
