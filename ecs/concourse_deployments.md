# Service check after a deployment of an ECS service with Concourse

Out of the box Terraform does not wait for the ECS service to be in service. To make sure that we get notified if a deploy succeeded without the need of logging into aws we need to add the following task after the deployment to check the status of the service:

```yaml
---
platform: linux
image_resource:
  type: docker-image
  source:
    repository: "skyscrapers/awscli"
    tag: "latest"

params:
  AWS_ACCOUNT_ID: ''
  AWS_ACCESS_KEY_ID: ''
  AWS_SECRET_ACCESS_KEY: ''
  AWS_DEFAULT_REGION: 'eu-west-1'
  ECS_CLUSTER_NAME: ''
  ECS_SERVICE_NAME: ''

run:
  path: sh
  args:
  - -ec
  - |
    # Assume Role in the ops AWS account
    ASSUMED_ROLE=$(aws sts assume-role --role-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/ops/admin --role-session-name concourse-$BUILD_PIPELINE_NAME-$BUILD_JOB_NAME)
    export AWS_ACCESS_KEY_ID=$(echo "$ASSUMED_ROLE" | jq -r ".Credentials.AccessKeyId")
    export AWS_SECRET_ACCESS_KEY=$(echo "$ASSUMED_ROLE" | jq -r ".Credentials.SecretAccessKey")
    export AWS_SESSION_TOKEN=$(echo "$ASSUMED_ROLE" | jq -r ".Credentials.SessionToken")

    # Check service status
    echo "Checking $ECS_SERVICE_NAME deployment status..."
    if `aws  ecs wait services-stable --cluster $ECS_CLUSTER_NAME --services $ECS_SERVICE_NAME`
    then
        echo "$ECS_SERVICE_NAME reached a successful state"
    else
        echo "$ECS_SERIVCE_NAME failed to reach stable state or the check has been timed out"
        exit 1
    fi
```

This task first gets the needed credentials and then connects to the aws api to check the status of the service. If the deployment succeeded the command will return 0. If the service doesn't come up during the deployment the task will return status code 255 and will show up as a failure in Concourse.
