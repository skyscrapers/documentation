# AWS Service Operator

The AWS Service Operator allows you to manage some AWS resources, like [ECR repositrories](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Repositories.html) and [S3 buckets](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html), by using Kubernetes [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

**WARNING**: This is considered alpha software and we don't enable this feature by default, due to very broad IAM permission requirements. If you would like to use the AWS Service Operator, please create a GH issue or get into contact with your lead to enable it.

## Basic usage

Create an ecr repository:

```yaml
apiVersion: service-operator.aws/v1alpha1
kind: ECRRepository
metadata:
  name: my-ecr-repo
```

Create an S3 bucket:

```yaml
apiVersion: service-operator.aws/v1alpha1
kind: S3Bucket
metadata:
  name: my-private-bucket
spec:
  versioning: true
  accessControl: Private
```

You can find more examples in the [upstream documentation](https://github.com/awslabs/aws-service-operator/tree/master/examples)

## IAM permissions on created resources

Unfurtunately you can't really control IAM permissions yet via the Operator. This means your app might not have access (through [kube2iam](walk-through.md#IAM-Roles)) to eg. the S3 bucket you created via the Operator.

One workaround to keep permissions still dynamic, is that you make sure your Operator-managed resources follow a consistent naming pattern and that we create a `kube2iam` role which applies to this pattern.
