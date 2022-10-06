<!-- markdownlint-disable no-inline-html -->

# Github Actions runner controller

You can dynamically deploy Github self-hosted runners on the Kubernetes clusters using the [`actions-runner-controller`](https://github.com/actions-runner-controller/actions-runner-controller). The controller allows developers to define either organization or repository runners, and it automatically handles the registration and de-registration of those.

The controller is disabled by default but we can quickly enable it upon request.

## Github authentication

In order for the controller to authenticate to Github and manage the runners, it needs either a Github application or a Github Personal Access Token (PAT) with access to the Github organization. We highly encourage you to use the Github application method, as the PAT requires very broad and high level access to the Github organization, which poses a high security risk would the token be exposed. You can read more about how the controller authenticates with Github in the [official documentation](https://github.com/actions-runner-controller/actions-runner-controller#setting-up-authentication-with-github-api).

To configure the `actions-runner-controller` with a Github application follow these steps:

*Note that these steps are taken [from the controller main documentation](https://github.com/actions-runner-controller/actions-runner-controller#deploying-using-github-app-authentication), and tailored to our setup to make it easier for our customers*

1. Create a new Github App for your organization by following this URL (replace `:org` with your organization name):

   ```text
   https://github.com/organizations/:org/settings/apps/new?url=http://github.com/actions-runner-controller/actions-runner-controller&webhook_active=false&public=false&administration=write&organization_self_hosted_runners=write&actions=read&checks=read
   ```

    You will create a Github App with the following permissions (already enabled if you've followed the link above):
    - Required Permissions for Repository Runners:
      - Repository Permissions
        - Actions (read)
        - Administration (read / write)
        - Checks (read) (to use [Webhook Driven Scaling](https://github.com/actions-runner-controller/actions-runner-controller#webhook-driven-scaling))
        - Metadata (read)
    - Required Permissions for Organization Runners:
        - Repository Permissions
          - Actions (read)
          - Metadata (read)
        - Organization Permissions
          - Self-hosted runners (read / write)

    *Note: All API routes mapped to their permissions can be found [here](https://docs.github.com/en/rest/reference/permissions-required-for-github-apps) if you wish to review*

2. You will see an *App ID* on the page of the GitHub App you created as follows, the value of this App ID will be used later.
    <img width="750" alt="App ID" src="https://user-images.githubusercontent.com/230145/78968802-6e7c8880-7b40-11ea-8b08-0c1b8e6a15f0.png">
3. Download the private key file by pushing the "Generate a private key" button at the bottom of the GitHub App page. This file will also be used later.
    <img width="750" alt="Generate a private key" src="https://user-images.githubusercontent.com/230145/78968805-71777900-7b40-11ea-97e6-55c48dfc44ac.png">
4. Go to the "Install App" tab on the left side of the page and install the GitHub App that you created for your account or organization.
    <img width="750" alt="Install App" src="https://user-images.githubusercontent.com/230145/78968806-72100f80-7b40-11ea-810d-2bd3261e9d40.png">
5. When the installation is complete, you will be taken to a URL in one of the following formats, the last number of the URL will be used as the Installation ID later (For example, if the URL ends in `settings/installations/12345`, then the Installation ID is `12345`).

    - `https://github.com/settings/installations/${INSTALLATION_ID}`
    - `https://github.com/organizations/eventreactor/settings/installations/${INSTALLATION_ID}`

6. Finally, register the App ID (`APP_ID`), Installation ID (`INSTALLATION_ID`), and downloaded private key file (`PRIVATE_KEY_FILE_PATH`) to Kubernetes as `Secret` in the `github-actions-runner-system` namespace (**remember to set the correct kubectl context before running this command**):

    ```shell
    $ kubectl create secret generic github-actions-controller-manager \
        -n github-actions-runner-system \
        --from-literal=github_app_id=${APP_ID} \
        --from-literal=github_app_installation_id=${INSTALLATION_ID} \
        --from-file=github_app_private_key=${PRIVATE_KEY_FILE_PATH}
    ```

After creating the `Secret` you'll be ready to start setting up runners via the `actions-runner-controller` CRDs (`Runner`, `RunnerDeployment`, ...). Head to [the controller documentation](https://github.com/actions-runner-controller/actions-runner-controller/blob/master/README.md#usage) to know how to define your runners.

Note that `Runner` are "single-job-run" setups, meaning that once the runner has processed a single job, it will de-register itself and exit, and the controller won't replace it. If you want a long-lasting runner setup that is there to process any incoming jobs, you'll need to setup either a [`RunnerDeployment`](https://github.com/actions-runner-controller/actions-runner-controller/blob/master/docs/detailed-docs.md#runnerdeployments) or a [`RunnerSet`](https://github.com/actions-runner-controller/actions-runner-controller/blob/master/docs/detailed-docs.md#runnersets). With those, the controller will make sure there's always the configured number of active runners listening for new jobs.

You can deploy your runners in any namespace on the cluster where you have access to. The runner Pod(s) will be created in the same namespace where you create the `RunnerSet` or `RunnerDeployment` resources.

Note that currently our setup doesn't support runner autoscaling. Let us know if that would be useful for you and we'll push it to our platform team to get it implemented.
