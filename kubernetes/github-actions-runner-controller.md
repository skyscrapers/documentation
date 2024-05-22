<!-- markdownlint-disable no-inline-html -->

# Github Actions runner controller

You can dynamically deploy Github self-hosted runners on the Kubernetes clusters using the [`actions-runner-controller`](https://github.com/actions-runner-controller/actions-runner-controller). The controller allows developers to define either organization or repository runners, and it automatically handles the registration and de-registration of those.

The controller is disabled by default but we can quickly enable it upon request.

## Github authentication

In order for the controller to authenticate to Github and manage the runners, it needs either a Github application or a Github Personal Access Token (PAT) with access to the Github organization. We highly encourage you to use the Github application method, as the PAT requires very broad and high level access to the Github organization, which poses a high security risk would the token be exposed. You can read more about how the controller authenticates with Github in the [official documentation](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api).

To configure the `actions-runner-controller` with a Github application follow these steps:

> [!NOTE]
> These steps are taken [from the controller main documentation](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api#authenticating-arc-with-a-github-app), and tailored to our setup to make it easier for you to follow.

1. Create a new Github App for your org anization by following this URL (replace `:org` with your organization name):

   ```text
   https://github.com/organizations/:org/settings/apps/new?url=http://github.com/actions-runner-controller/actions-runner-controller&webhook_active=false&public=false&administration=write&organization_self_hosted_runners=write&actions=read&checks=read
   ```

    You will create a Github App with the following permissions (already enabled if you've followed the link above):
    - Required Permissions for Repository Runners:
      - Repository Permissions
        - Actions (read)
        - Administration (read / write)
        - Checks (read) (to use [Webhook Driven Scaling](https://github.com/actions/actions-runner-controller/blob/master/docs/automatically-scaling-runners.md))
        - Metadata (read)
    - Required Permissions for Organization Runners:
        - Repository Permissions
          - Actions (read)
          - Metadata (read)
        - Organization Permissions
          - Self-hosted runners (read / write)

   > [!NOTE]
   > All API routes mapped to their permissions can be found [here](https://docs.github.com/en/rest/reference/permissions-required-for-github-apps) if you wish to review*

2. You will see an *App ID* on the page of the GitHub App you created as follows, the value of this App ID will be used later.
    <img width="750" alt="App ID" src="https://user-images.githubusercontent.com/230145/78968802-6e7c8880-7b40-11ea-8b08-0c1b8e6a15f0.png">
3. Download the private key file by pushing the "Generate a private key" button at the bottom of the GitHub App page. This file will also be used later.
    <img width="750" alt="Generate a private key" src="https://user-images.githubusercontent.com/230145/78968805-71777900-7b40-11ea-97e6-55c48dfc44ac.png">
4. Go to the "Install App" tab on the left side of the page and install the GitHub App that you created for your account or organization.
    <img width="750" alt="Install App" src="https://user-images.githubusercontent.com/230145/78968806-72100f80-7b40-11ea-810d-2bd3261e9d40.png">
5. When the installation is complete, you will be taken to a URL in one of the following formats, the last number of the URL will be used as the Installation ID later (For example, if the URL ends in `settings/installations/12345`, then the Installation ID is `12345`).

    - `https://github.com/settings/installations/${INSTALLATION_ID}`
    - `https://github.com/organizations/eventreactor/settings/installations/${INSTALLATION_ID}`

6. Finally, register the App ID (`APP_ID`), Installation ID (`INSTALLATION_ID`), and downloaded private key file (`PRIVATE_KEY_FILE_PATH`) to Kubernetes as `Secret` in the `arc-system` namespace (**remember to set the correct kubectl context before running this command**):

    ```shell
    $ kubectl create secret generic github-actions-controller-manager \
        -n arc-system \
        --from-literal=github_app_id=${APP_ID} \
        --from-literal=github_app_installation_id=${INSTALLATION_ID} \
        --from-file=github_app_private_key=${PRIVATE_KEY_FILE_PATH}
    ```

Alternatively, you can write the Github app credentials in a secret in AWS Secrets Manager, and configure ARC accordingly to fetch the details from there. Note that you'll need to have the AWS Secrets Manager CSI driver enabled in your cluster. If you want to take this route, omit the last step (6.) above, and instead create a secret in AWS Secrets Manager following this format:

```json
{
    "github_app_id": "",
    "github_app_installation_id": "",
    "github_app_private_key": ""
}
```

After the secret is updated and contains the necessary info, update your cluster definition file as follows:

```yaml
...
spec:
  github_actions_runner_controller:
    enabled: true
    aws_secrets_manager:
      secret_arn: <secret-arn>
```

After authentication is configured you'll be ready to start setting up [the runners](https://github.com/skyscrapers/kubernetes-stack/blob/main/modules/gha-runner-scale-set/README.md). This is done by Skyscrapers with Terragrunt/OpenTofu:

   ```hcl
   module "runner_scale_set" {
     source      = "git::ssh://git@github.com/skyscrapers/kubernetes-stack//modules/gha-runner-scale-set?ref=main"
     environment = var.environment

     eks_cluster_oidc_provider_name = local.eks_cluster_oidc_provider_name
     eks_cluster_oidc_provider_arn  = local.eks_cluster_oidc_provider_arn

     github_config_url = "https://github.com/<organisation-id>>"

     max_runners = 5
     min_runners = 1 # This is the minimum number of runners that will be kept alive (can scale to 0 if no jobs are running)

     runner_scale_set_name = "k8s-runners-${var.environment}"

     gha_admin_k8s_namespaces = [${var.environment}]

     dind_enabled = false

     cpu    = "1"
     memory = "1.5Gi"
   }
   ```

Once those runners are up and running you can verify the runners are online in the GitHub UI and you can update the jobs in your workflows to use the runners by setting the `runs-on:` key in the job definition to the name of the runner you want to use (eg `kubernetes-runners-<environment>` in this example).

> [!TIP]
> This module also outputs the Service Account and IAM Role that the runners are using, so those can be used to grant the necessary permissions to the runners.
