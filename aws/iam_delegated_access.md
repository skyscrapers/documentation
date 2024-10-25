# How to provide Skyscrapers access to your AWS account

## Overview

In order to provide Skyscrapers with access to your AWS account(s), you can use this documentation to configure your accounts with the necessary permissions and trust relationship towards Skyscrapers. This will allow Skyscrapers engineers to assume these roles and perform the necessary tasks in your account.

The following roles can be created:

- The [ReadOnly IAM role](./cloudformation_templates/sks_ro.yml) gets deployed with the `ReadOnlyAccess` policy. This allows Skyscrapers engineers to obtain read-only access in the AWS account. This role is useful for monitoring, auditing, and troubleshooting purposes.
- [Admin IAM role](./cloudformation_templates/sks_admin.yml) gets deployed with the `AdministratorAccess` policy. This allows Skyscrapers engineers to obtain full access in the AWS account. This role is useful for performing administrative tasks, such as creating, modifying, and deleting resources.

This guide will walk you through the steps to create an ReadOnly and Admin IAM role in your AWS account through CloudFormation.

## Prerequisites

- You must have an AWS account with sufficient permissions to create IAM roles and apply CloudFormation stacks.
- Familiarity with CloudFormation and AWS IAM concepts is recommended but not required.

## Steps to Apply the Template

1. **Download the CloudFormation Template**

Download the CloudFormation YAML template file(s) you want to apply using the following link or copy the content below:

- [ReadOnly IAM role](./cloudformation_templates/sks_ro.yml)
- [Admin IAM role](./cloudformation_templates/sks_admin.yml)

### Deploy the Template via the AWS Management Console

> [!NOTE]
> If you encounter any issues or have questions, please contact the Skyscrapers support team for assistance.

1. Log in to the [AWS Management Console](https://aws.amazon.com/console/).
2. Navigate to **CloudFormation** using the search bar or [click here](https://eu-west-1.console.aws.amazon.com/cloudformation/home).
   ![CloudFormation](./img/CF_home.png)
3. Click **Create stack** if this is your first stack or alternatively at the top right and select **With new resources (standard)**.
4. In the **Create Stack** page:
   - Choose **Upload a template file** and upload the downloaded YAML file.
   - Click **Next**.
   ![Create Stack](./img/step1.png)
5. Provide a **Stack Name** (e.g., `Skyscrapers-Readonly-Access`) and click **Next** to proceed.
   ![Stack Name](./img/step2.png)
6. Under **Configure stack options**, you can add stack-level tags if desired and click **Next** to proceed.
   - You'll need to acknowledge that the template may create IAM resources as the template creates IAM roles.
   ![Stack Configuration](./img/step3.png)
7. Review your stack configuration, and check the box acknowledging the creation of IAM resources.
   ![Review](./img/step4.png)
8. Click **Create stack** to start the deployment.
9. Verify the deployment status in the CloudFormation console.
   ![Deployment](./img/deploy.png)

Once the deployment is complete, navigate to the **CloudFormation** console and ensure that the status of your stack shows **CREATE_COMPLETE**.

To verify the created role:

1. Go to the **IAM Console**.
2. Under **Roles**, find the role named `sks-readonly` and/or `sks-admin`.
3. Review the trust relationship and the attached policies to ensure everything is set up correctly.

## Updating or Deleting the Stack

- To **update** the stack, you can re-upload a modified version of the template in the CloudFormation console and choose **Update Stack**.
- To **delete** the stack, navigate to the CloudFormation console, select the stack, and click **Delete**.

## Access to Billing & Costs

> [!CAUTION]
> Enabling **IAM User and Role Access To Billing Information** will allow existing users/roles access to the billing dashboard **IF** you those users have a full read-access '*' on all resources.

To allow us access to the billing dashboard, it first needs to be enabled.

If you're using an AWS organisation, then it needs to be done via the Master Payer Account.

Please do the following steps to give us that access:

1. Sign in to the AWS Management Console with your root user credentials (specifically, the email address and password that you used to create your AWS account).
2. On the navigation bar, select your account name, and then select [Account](https://console.aws.amazon.com/billing/home#/account).
3. Scroll down the page until you find the section **IAM User and Role Access to Billing Information**, then click Edit.
4. Select the **Activate IAM Access** check box to activate access to the Billing and Cost Management console pages.
5. Choose Update.
