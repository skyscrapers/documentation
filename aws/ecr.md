# Amazon Elastic Container Registry (ECR)

## Introduction

We usually recommend and setup [ECR](https://aws.amazon.com/ecr/) for our customers.

AWS documentation: <https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html>

## Setup cross-account ECR access

> [!NOTE]
> Primary audience: Skyscrapers internal

Our blueprints seperate different environments into different AWS accounts (eg. `CustomerStaging`, `CustomerProduction`, `CustomerSharedTooling`, ...). We also recommend to only build a single artifact (container in this case) which is used throughout all environments, meaning ECR repositories are usually hosted in the `CustomerSharedTooling` account.

To provide environment access to these images, we need to setup cross-account access to ECR repositories. On environment account side, our Kubernetes stack codebases already handle the necessary permissions (EKS worker nodes, Flux source controller), but we also need to setup the necessary permissions on the ECR repository side.

This can be done as follows:

```terraform
data "aws_iam_policy_document" "cross_account" {
  statement {
    sid    = "ECRCrossAccountAccess"
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = var.cross_account_principal_identifiers
    }

    actions = [
      "ecr:GetAuthorizationToken",
      "ecr:BatchCheckLayerAvailability",
      "ecr:GetDownloadUrlForLayer",
      "ecr:GetRepositoryPolicy",
      "ecr:DescribeRepositories",
      "ecr:ListImages",
      "ecr:DescribeImages",
      "ecr:BatchGetImage",
      "ecr:DescribeImageScanFindings"
    ]
  }
}

resource "aws_ecr_repository_policy" "cross_account" {
  for_each   = aws_ecr_repository.repo

  repository = each.value.name
  policy     = data.aws_iam_policy_document.cross_account.json
}
```

Where `var.cross_account_principal_identifiers` is a list of AWS principal identifiers which should have access to the ECR repositories. Usually this is eg. `arn:aws:iam::123456789012:root` for granting a whole AWS account access, but you can also be more specific like `arn:aws:iam::123456789012:role/development-eks-example-com-workers`.

One caveat is that AWS performs a check whether the target principal exists when deploying the policy, so make sure that those target-side roles are already created before applying this policy Before you save the repository policy, make sure that the role exists in the secondary account. If the role doesn't exist, you'll receive an error similar to `invalid repository policy provided`.
