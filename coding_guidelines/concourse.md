# Concourse

Here are some recommendations and best practices for writing Concourse pipeline and task definitions.

**TIP**: Pivotal has a [repository with Concourse pipeline samples and recipes](https://github.com/pivotalservices/concourse-pipeline-samples), which is very useful and comprehensive.

## Naming

* Resource names should contain a reference to the resource type at the end of the name. Resources don't have any explicit reference to the type of resource they are, so it's a good practice to put that in the name. Examples:
  * `teleport-server-stack-git` is a [`git` resource](https://github.com/concourse/git-resource)
  * `teleport-server-ami` is an [`ami` resource](https://github.com/jdub/ami-resource)

## Linting

You can run `fly validate-pipeline -c pipeline.yaml` to lint your pipeline definitions and search for errors and warnings. You also have to provide any parameters file your pipeline needs, with `-l`.

## Formatting

The only formatting rule enforced at the moment is to use **two spaces for indentation**.

**TIP**: You can also run `fly format-pipeline -c pipeline.yaml` to format a pipeline config in a "canonical" form (i.e. keys are in normal order, with name first for example)

## Reusability

Here are some recommendations for better code reusability:

* Use [task files](https://concourse-ci.org/tasks.html) whenever possible, instead of defining your tasks in-line in your pipeline definitions. Task files allow you to reuse your tasks in a pipeline, or even across multiple pipelines.
  * Try to **avoid multi-purpose tasks**. Tasks are the smallest configurable unit in a Concourse pipeline, and they should be seen as software functions. So it's a good practice to write your task definitions as atomic as possible. For example, you would have a task for running your tests, one for linting and another one for deploying.
  * Make your tasks as **generic** as possible, to increase re-usability. Task [`params`](https://concourse-ci.org/tasks.html#task-params) will help you on that.
  * Create a **docker image** for your tasks, with all dependencies baked-in. Tasks run inside docker containers, and having to install all the external dependencies on every run is very time-consuming and not very efficient. For example, if your task uses Terraform, create a docker image with Terraform pre-installed, and run your task on that image.
* Use [YAML node anchors](https://en.wikipedia.org/wiki/YAML#Advanced_components) to reuse blocks in your pipeline definitions. Avoid using [pipeline `\((params))`](https://concourse-ci.org/setting-pipelines.html) for replicating code blocks. For example:
  * **Good**: everything in the **pipeline definition** using YAML node anchors

    In `pipeline.yaml`:

    ```yaml
    shared:
    - &failure-alert
      put: slack-alert
      params:
        silent: true
        attachments:
        - color: danger
          text: |
            The <$ATC_EXTERNAL_URL/teams/main/pipelines/$BUILD_PIPELINE_NAME/jobs/$BUILD_JOB_NAME/builds/$BUILD_NAME|$BUILD_PIPELINE_NAME - $BUILD_JOB_NAME> job failed!

    ...

    jobs:
    - name: build
      on_failure: *failure-alert
      plan:
        - get: skybot-repo
          trigger: true
    ...
    ```

  * **Bad**: having a separated `common.yaml` file with pipeline `\((params))` to re-use code blocks.

    In `common.yaml`

    ```yaml
    SLACK_ON_FAILURE_ALERT:
      put: slack-alert
      params:
        silent: true
        attachments:
        - color: danger
          text: |
            The <$ATC_EXTERNAL_URL/teams/main/pipelines/$BUILD_PIPELINE_NAME/jobs/$BUILD_JOB_NAME/builds/$BUILD_NAME|$BUILD_PIPELINE_NAME - $BUILD_JOB_NAME> job failed!
    ```

    In `pipeline.yaml`:

    ```yaml
    jobs:
    - name: build
      on_failure: ((SLACK_ON_FAILURE_ALERT))
      plan:
        - get: skybot-repo
          trigger: true
    ...
    ```

* Write generic pipeline definitions whenever possible. Sometimes a Concourse pipeline will be very application and/or business specific, but other pipelines might be to test and deploy standard components or applications. In the later case, you could write the pipeline definition as generic as possible, leaving all custom values (like environment names or git repo urls) as [pipeline `\((params))`](https://concourse-ci.org/setting-pipelines.html). An example can be found in the [teleport-server-stack](https://github.com/skyscrapers/teleport-server-stack#concourse-pipeline) component.
* **Use Concourse [resources](https://concourse-ci.org/resources.html)** instead of tasks whenever possible. For example, to build a packer image, you could write a task that runs packer, or you could use the [packer resource](https://github.com/jdub/packer-resource). Using the resource is preferred, as it integrates better with the rest of the pipeline. Following the previous example, when using the packer resource you can easily trigger other jobs in the pipeline when the image has finished building.
  * List of the resource types provided by Concourse: <https://concourse-ci.org/included-resources.html>
  * List of third-party community resource types: <https://concourse-ci.org/community-resources.html>

## File structure

In Concourse, there are basically two types of files: task files and pipeline files.

* Conventionally a task's configuration is placed in the same repository as the code it's acting on, normally under the `/ci` directory.
* Our pipeline definitions are in the [`ci` repo](https://github.com/skyscrapers/ci).
* It's not yet entirely clear where to put the pipeline definitions for standard components, like the one for the [teleport-server-stack](https://github.com/skyscrapers/teleport-server-stack#concourse-pipeline). For now they are stored together with the standard component that they represent.
* Our customers are free to put their pipeline definitions wherever they see fit, although we recommend them to put them all in a single place, for easier access. It's also recommended that the customer documentation has a reference to where they store their pipeline definitions.

## Secrets management

* Use the [Vault integration](https://github.com/skyscrapers/concourse-stack/#vault) whenever possible. This is the preferred and more secure method to use sensitive information in your pipelines, as Concourse doesn't store that information anywhere, so it can't be retrieved via `fly get-pipeline -p ...`.
* If the Vault integration is not an option, use [pipeline `\((params))`](https://concourse-ci.org/setting-pipelines.html) to set sensitive information in your pipelines. This way you keep the sensitive information separated from your pipeline definition, so you can put the pipeline definition in source control, but not your secrets. Using this method, secrets can still be retrieved from concourse by fetching the pipeline definition with `fly get-pipeline -p ...`.
  * Distribute the secret parameters file in a secure way, and never commit it in source control.

## AWS credentials

In order to authenticate your jobs and resources to AWS, you must use individual IAM users for each purpose, with fine-grained permissions, and set the credentials for those users in your pipelines as secrets.

**IMPORTANT**: You shouldn't use the Concourse worker instance profile to authenticate your jobs to AWS.

The main reason for this is that Concourse is a multi-tenant setup, so by using the worker instance profile you would be giving the full set of permissions to all the jobs and teams running in Concourse, which poses a security risk. By using individual IAM users you can give much more fine-grained permissions to each job, pipeline and team.
Also, the AWS credentials for those IAM users can be securely stored in Vault. In the future, when Concourse supports it, we'll be using [Vault's AWS secrets engine](https://www.vaultproject.io/docs/secrets/aws/index.html) to dynamically generate individual short-lived AWS credentials for each job.
