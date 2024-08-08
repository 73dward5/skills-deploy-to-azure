<header>

<!--
  <<< Author notes: Course header >>>
  Include a 1280x640 image, course title in sentence case, and a concise description in emphasis.
  In your repository settings: enable template repository, add your 1280x640 social image, auto delete head branches.
  Add your open source license, GitHub uses MIT license.
-->

# Deploy to Azure

_Create two deployment workflows using GitHub Actions and Microsoft Azure._

</header>

<!--
  <<< Author notes: Step 5 >>>
  Start this step by acknowledging the previous step.
  Define terms and link to docs.github.com.
-->

<<<<<<< HEAD
## Step 4: Deploy to a staging environment based on labels

_Nicely done, you used a workflow to spin up your Azure environment :dancer:_

Now that the proper configuration and workflow files are present, let's test our actions! In this step, there's a small change to the game. Once you add the appropriate label to your pull request, you should be able to see the deployment!

1. Create a new branch named `staging-test` from `main` using the same steps as you did for the previous `azure-configuration` branch.
1. Edit the `.github/workflows/deploy-staging.yml` file, and replace every `<username>` with your GitHub username.
1. Commit that change to the new `staging-test` branch.
1. Go to the Pull requests tab and there should be a yellow banner with the `staging-test` branch to `Compare & pull request`. Once the pull request is opened up, click `Create pull request`.

### :keyboard: Activity 1: Add the proper label to your pull request

<<<<<<< HEAD
Through the power of GitHub Actions, we can create, configure, and destroy these resources through our workflow files.

### :keyboard: Activity 1: Set up a personal access token (PAT)

Personal access tokens (PATs) are an alternative to using passwords for authentication to GitHub. We will use a PAT to allow your web app to pull the container image after your workflow pushes a newly built image to the registry.

1. Open a new browser tab, and work on the steps in your second tab while you read the instructions in this tab.
2. Create a personal access token with the `repo` and `write:packages` scopes. For more information, see ["Creating a personal access token."](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
3. Once you have generated the token we will need to store it in a secret so that it can be used within a workflow. Create a new repository secret named `CR_PAT` and paste the PAT token in as the value.
4. With this done we can move on to setting up our workflow.

**Configuring your Azure environment**

To deploy successfully to our Azure environment:

1. Create a new branch called `azure-configuration` by clicking on the branch dropdown on the top, left hand corner of the `Code` tab on your repository page.
2. Once you're in the new `azure-configuration` branch, go into the `.github/workflows` directory and create a new file titled `spinup-destroy.yml` by clicking **Add file**. Copy and paste the following into this new file:
    ```yaml
    name: Configure Azure environment

    on:
      pull_request:
        types: [labeled]

    env:
      IMAGE_REGISTRY_URL: ghcr.io
      AZURE_RESOURCE_GROUP: cd-with-actions
      AZURE_APP_PLAN: actions-ttt-deployment
      AZURE_LOCATION: '"East US"'
      ###############################################
      ### Replace <username> with GitHub username ###
      ###############################################
      AZURE_WEBAPP_NAME: 73dward5-ttt-app

    jobs:
      setup-up-azure-resources:
        runs-on: ubuntu-latest
        if: contains(github.event.pull_request.labels.*.name, 'spin up environment')
        steps:
          - name: Checkout repository
            uses: actions/checkout@v4

          - name: Azure login
            uses: azure/login@v2
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}

          - name: Create Azure resource group
            if: success()
            run: |
              az group create --location ${{env.AZURE_LOCATION}} --name ${{env.AZURE_RESOURCE_GROUP}} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}

          - name: Create Azure app service plan
            if: success()
            run: |
              az appservice plan create --resource-group ${{env.AZURE_RESOURCE_GROUP}} --name ${{env.AZURE_APP_PLAN}} --is-linux --sku F1 --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}

          - name: Create webapp resource
            if: success()
            run: |
              az webapp create --resource-group ${{ env.AZURE_RESOURCE_GROUP }} --plan ${{ env.AZURE_APP_PLAN }} --name ${{ env.AZURE_WEBAPP_NAME }}  --deployment-container-image-name nginx --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}

          - name: Configure webapp to use GHCR
            if: success()
            run: |
              az webapp config container set --docker-custom-image-name nginx --docker-registry-server-password ${{secrets.CR_PAT}} --docker-registry-server-url https://${{env.IMAGE_REGISTRY_URL}} --docker-registry-server-user ${{github.actor}} --name ${{ env.AZURE_WEBAPP_NAME }} --resource-group ${{ env.AZURE_RESOURCE_GROUP }} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}}

      destroy-azure-resources:
        runs-on: ubuntu-latest

        if: contains(github.event.pull_request.labels.*.name, 'destroy environment')

        steps:
          - name: Checkout repository
            uses: actions/checkout@v4

          - name: Azure login
            uses: azure/login@v2
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}

          - name: Destroy Azure environment
            if: success()
            run: |
              az group delete --name ${{env.AZURE_RESOURCE_GROUP}} --subscription ${{secrets.AZURE_SUBSCRIPTION_ID}} --yes
    ```
1. Click **Commit changes...** and select `Commit directly to the azure-configuration branch.` before clicking **Commit changes**.
1. Go to the Pull requests tab of the repository.
1. There should be a yellow banner with the `azure-configuration` branch where you can click **Compare & pull request**.
1. Set the title of the Pull request to: `Added spinup-destroy.yml workflow` and click `Create pull request`.

We will cover the key functionality below and then put the workflow to use by applying a label to the pull request.

This new workflow has two jobs:

1. **Set up Azure resources** will run if the pull request contains a label with the name "spin up environment".
2. **Destroy Azure resources** will run if the pull request contains a label with the name "destroy environment".

In addition to each job, there's a few global environment variables:

- `AZURE_RESOURCE_GROUP`, `AZURE_APP_PLAN`, and `AZURE_WEBAPP_NAME` are names for our resource group, app service plan, and web app, respectively, which we'll reference over multiple steps and workflows
- `AZURE_LOCATION` lets us specify the [region](https://azure.microsoft.com/en-us/global-infrastructure/regions/) for the data centers, where our app will ultimately be deployed.

**Setting up Azure resources**

The first job sets up the Azure resources as follows:

1. Logs into your Azure account with the [`azure/login`](https://github.com/Azure/login) action. The `AZURE_CREDENTIALS` secret you created earlier is used for authentication.
1. Creates an [Azure resource group](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/overview#resource-groups) by running [`az group create`](https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create) on the Azure CLI, which is [pre-installed on the GitHub-hosted runner](https://help.github.com/en/actions/reference/software-installed-on-github-hosted-runners).
1. Creates an [App Service plan](https://docs.microsoft.com/en-us/azure/app-service/overview-hosting-plans) by running [`az appservice plan create`](https://docs.microsoft.com/en-us/cli/azure/appservice/plan?view=azure-cli-latest#az-appservice-plan-create) on the Azure CLI.
1. Creates a [web app](https://docs.microsoft.com/en-us/azure/app-service/overview) by running [`az webapp create`](https://docs.microsoft.com/en-us/cli/azure/webapp?view=azure-cli-latest#az-webapp-create) on the Azure CLI.
1. Configures the newly created web app to use [GitHub Packages](https://help.github.com/en/packages/publishing-and-managing-packages/about-github-packages) by using [`az webapp config`](https://docs.microsoft.com/en-us/cli/azure/webapp/config?view=azure-cli-latest) on the Azure CLI. Azure can be configured to use its own [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/), [DockerHub](https://docs.docker.com/docker-hub/), or a custom (private) registry. In this case, we'll configure GitHub Packages as a custom registry.

**Destroying Azure resources**

The second job destroys Azure resources so that you do not use your free minutes or incur billing. The job works as follows:

1. Logs into your Azure account with the [`azure/login`](https://github.com/Azure/login) action. The `AZURE_CREDENTIALS` secret you created earlier is used for authentication.
1. Deletes the resource group we created earlier using [`az group delete`](https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-delete) on the Azure CLI.

### :keyboard: Activity 2: Apply labels to create resources

1. Edit the `spinup-destroy.yml` file in your open pull request and replace any `<username>` placeholders with your GitHub username. Commit this change directly to the `azure-configuration` branch.
1. Back in the Pull request, create and apply the `spin up environment` label to your open pull request
1. Wait for the GitHub Actions workflow to run and spin up your Azure environment. You can follow along in the Actions tab or in the pull request merge box.
=======
1. Ensure that the `GITHUB_TOKEN` for this repository has read and write permissions under **Workflow permissions**. [Learn more](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#modifying-the-permissions-for-the-github_token). This is required for your workflow to be able to upload your image to the container registry.
1. Create and apply the `stage` label to your open pull request
1. Wait for the GitHub Actions workflow to run and deploy the application to your Azure environment. You can follow along in the Actions tab or in the pull request merge box. The deployment may take a few moments but you've done the right thing. Once the deployment is successful, you'll see green check marks for each run, and you'll see a URL for your deployment. Play the game!
>>>>>>> main
=======
## Step 5: Deploy to a production environment based on labels

_Deployed! :ship:_

### Nicely done

As we've done before, create a new branch called `production-deployment-workflow` from `main`. In the `.github/workflows` directory, add a new file titled `deploy-prod.yml`. This new workflow deals specifically with commits to `main` and handles deployments to `prod`.

**Continuous delivery** (CD) is a concept that contains many behaviors and other, more specific concepts. One of those concepts is **test in production**. That can mean different things to different projects and different companies, and isn't a strict rule that says you are or aren't "doing CD".

In our case, we can match our production environment to be exactly like our staging environment. This minimizes opportunities for surprises once we deploy to production.

### :keyboard: Activity 1: Add triggers to production deployment workflow

Copy and paste the following to your file, and replace any `<username>` placeholders with your GitHub username. Note that not much has changed from our staging workflow, except for our trigger, and that we won't be filtering by labels.

```yaml
name: Deploy to production

on:
  push:
    branches:
      - main

env:
  IMAGE_REGISTRY_URL: ghcr.io
  ###############################################
  ### Replace <username> with GitHub username ###
  ###############################################
  DOCKER_IMAGE_NAME: <username>-azure-ttt
  AZURE_WEBAPP_NAME: <username>-ttt-app
  ###############################################

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 16
      - name: npm install and build webpack
        run: |
          npm install
          npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: webpack artifacts
          path: public/

  Build-Docker-Image:
    runs-on: ubuntu-latest
    needs: build
    name: Build image and store in GitHub Container Registry
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Download built artifact
        uses: actions/download-artifact@v4
        with:
          name: webpack artifacts
          path: public

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.IMAGE_REGISTRY_URL }}
          username: ${{ github.actor }}
          password: ${{ secrets.CR_PAT }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{env.IMAGE_REGISTRY_URL}}/${{ github.repository }}/${{env.DOCKER_IMAGE_NAME}}
          tags: |
            type=sha,format=long,prefix=

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  Deploy-to-Azure:
    runs-on: ubuntu-latest
    needs: Build-Docker-Image
    name: Deploy app container to Azure
    steps:
      - name: "Login via Azure CLI"
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - uses: azure/docker-login@v1
        with:
          login-server: ${{env.IMAGE_REGISTRY_URL}}
          username: ${{ github.actor }}
          password: ${{ secrets.CR_PAT }}

      - name: Deploy web app container
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{env.AZURE_WEBAPP_NAME}}
          images: ${{env.IMAGE_REGISTRY_URL}}/${{ github.repository }}/${{env.DOCKER_IMAGE_NAME}}:${{github.sha}}

      - name: Azure logout via Azure CLI
        uses: azure/CLI@v2
        with:
          inlineScript: |
            az logout
            az cache purge
            az account clear
```

1. Update every `<username>` to your GitHub username.
1. Commit your changes to the `production-deployment-workflow` branch.
1. Go to the Pull requests tab and click **Compare & pull request** for the `production-deployment-workflow` branch and create a Pull request.

Great! The syntax you used tells GitHub Actions to only run that workflow when a commit is made to the main branch. Now we can put this workflow into action to deploy to production!

### :keyboard: Activity 2: Merge your pull request

1. You can now [merge](https://docs.github.com/en/get-started/quickstart/github-glossary#merge) your pull request!
1. Click **Merge pull request** and leave this tab open as we will be applying a label to the closed pull request in the next step.
1. Now we just have to wait for the package to be published to GitHub Container Registry and the deployment to occur.
>>>>>>> main
1. Wait about 20 seconds then refresh this page (the one you're following instructions from). [GitHub Actions](https://docs.github.com/en/actions) will automatically update to the next step.

<footer>

<!--
  <<< Author notes: Footer >>>
  Add a link to get support, GitHub status page, code of conduct, license link.
-->

---

Get help: [Post in our discussion board](https://github.com/orgs/skills/discussions/categories/deploy-to-azure) &bull; [Review the GitHub status page](https://www.githubstatus.com/)

&copy; 2023 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

</footer>
