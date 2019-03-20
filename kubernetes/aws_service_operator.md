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

## Overriding the default templates

The AWS Service Operator comes with a set of built-in Cloudformation templates, used to create the custom resources. It is possible to use your own templates by defining the corresponding `CloudFormationTemplate`.

For example if I want to create an S3Bucket template with some (hardcoded) CORS rules (see [caveats](#CloudFormationTemplate-limitations)):

```yaml
apiVersion: service-operator.aws/v1alpha1
kind: CloudFormationTemplate
metadata:
  name: s3bucket
data:
  key: s3bucket.yaml
  template: |
    AWSTemplateFormatVersion: 2010-09-09
    Description: 'AWS Service Operator - Amazon S3 Bucket (aso-1otq2cgce)'
    Parameters:
      Namespace:
        Description: >-
          This is the namespace for the Kubernetes object.
        Type: String
      ResourceVersion:
        Type: String
        Description: >-
          This is the resource version for the Kubernetes object.
      ResourceName:
        Description: >-
          This is the resource name for the Kubernetes object
        Type: String
      ClusterName:
        Description: >-
          This is the cluster name for the operator
        Type: String
      BucketName:
        Description: >-
          Must contain only lowercase letters, numbers, periods (.), and hyphens
          (-),Cannot end in numbers
        Type: String
        Default: apps3bucket
      BucketAccessControl:
        Description: define if the bucket can be accessed from public or private locations
        Type: String
        AllowedValues:
          - Private
          - PublicRead
          - PublicReadWrite
          - AuthenticatedRead
          - LogDeliveryWrite
          - BucketOwnerRead
          - BucketOwnerFullControl
          - AwsExecRead
        Default: "Private"
    Mappings: {}
    Resources:
      S3bucket:
        Type: 'AWS::S3::Bucket'
        Properties:
          BucketName: !Ref BucketName
          AccessControl: !Ref BucketAccessControl
          CorsConfiguration:
            CorsRules:
              - AllowedHeaders:
                  - "*"
                AllowedMethods:
                  - "GET"
                AllowedOrigins:
                  - "*"
          Tags:
            - Key: Namespace
              Value: !Ref Namespace
            - Key: ResourceVersion
              Value: !Ref ResourceVersion
            - Key: ResourceName
              Value: !Ref ResourceName
            - Key: ClusterName
              Value: !Ref ClusterName
            - Key: Heritage
              Value: operator.aws
        DeletionPolicy: Retain
    Outputs:
      BucketName:
        Value: !Ref S3bucket
        Description: Name of the sample Amazon S3 bucket.
      BucketArn:
        Value: !GetAtt
          - S3bucket
          - Arn
        Description: Name of the Amazon S3 bucket
```

You can find the default CloudFormationTemplates [in the upstream examples](https://github.com/awslabs/aws-service-operator/tree/master/examples/cloudformationtemplates).

## Caveats

Since the project is in early development, there are a lot of limitations / bugs / ... We've listed a couple of major ones here.

### IAM permissions on created resources

Unfurtunately you can't really control IAM permissions yet via the Operator. This means your app might not have access (through [kube2iam](walk-through.md#IAM-Roles)) to eg. the S3 bucket you created via the Operator.

One workaround to keep permissions still dynamic, is that you make sure your Operator-managed resources follow a consistent naming pattern and that we create a `kube2iam` role which applies to this pattern.

### CloudFormationTemplate limitations

You can override the default used Cloudformation templates via the `CloudFormationTemplate` CRD. However this comes with quite some limitations:

- You can only have 1 template per custom resource, eg. `s3bucket`. Adding different `CloudFormationTemplates` via namepaces will efectively overwrite the older one. FYI the actual template gets uploaded to an S3 bucket.
- There's no way yet to access parameters introduced in the `CloudFormationTemplate` from the matching custom resource. Thus any customization you add via the template will apply for any AWS resource you deploy with the CRD. [However a change for this seems to be WIP](https://github.com/awslabs/aws-service-operator/issues/140#issuecomment-454496454).
- Updating your `CloudFormationTemplate` doesn't trigger a change in resources already created before.
