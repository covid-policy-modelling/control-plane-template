# Covid Modeling Control Plane

Control plane environment where all model executions occur.

This model is meant to be used as a [template to create similar repositories](https://docs.github.com/en/free-pro-team@latest/github/creating-cloning-and-archiving-repositories/creating-a-repository-from-a-template) for use a control planes when executing models.

## Run Simulation

The [web-ui](https://github.com/covid-policy-modelling/web-ui) initiates model runs by sending a [repository_dispatch](https://docs.github.com/en/free-pro-team@latest/developers/webhooks-and-events/webhook-events-and-payloads#repository_dispatch) event to the control plane repo.
The configuration for the model run is passed as JSON in the `client_payload` portion of this event when it is triggered from the UI.

### Build Matrix

The [build matrix](https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/managing-complex-workflows#using-a-build-matrix) Actions functionality is used to run all of the models in parallel (if there is free runner capacity) as independent jobs.
Each model run can be tracked as an independent job within the workflow and will report back to and become available in the UI independently.

### Input File Templating and Generation

There is a small but important templating step in the workflow that uses `jq` to transform the list of models to run into a single model to run.
This is required for the matrix execution to work properly.
Note that the full field value is first extracted from the input payload in the matrix configuration.

### Artifact Collection

There are a number of job steps that can upload artifacts back to GitHub Actions to be stored with the job run.
Given that the data is copied by the model runner into an Azure Storage container, these steps are purely optional.
These are not run by default, but can be enabled through the use of secrets (below).

## Docker Compose

The `docker-compose.yaml` file is used as an easy way to store the complete configuration that is required to execute the [model-runner](https://github.com/covid-policy-modelling/model-runner) container.
The most import parts are the volume mounts for the input and output container as well as the Docker socket because the runner requires being able to use docker-in-docker.

## Self-hosted Runners

While it is possible to use the standard GitHub Action runners in order to execute the models, some models may require more resources (CPU and memory) than they have available.
As a result the example job is configured to use [self-hosted runners](https://docs.github.com/en/free-pro-team@latest/actions/hosting-your-own-runners/about-self-hosted-runners) so that the runners can be sure to have enough resources to run all the models.

Because we are mounting local storage in to the containers, it is very important that when using self-hosted runners the `Cleanup` step of the workflow is present and always executed.
This ensures that data from previous runs is not able to pollute the current run.

## Secrets

The workflow uses [secrets](https://docs.github.com/en/free-pro-team@latest/actions/reference/encrypted-secrets) as a mechanism to inject both credentials and configuration information would be burdensome or risky to store in the workflow file itself.

### Active Version

This is more of an environment variable than a secret, but allows for easily updating the version of the model runner that will be used during job execution.

```shell script
RUNNER_VERSION
```

#### Setting the active version of the model runner

1. Go to Settings > Secrets.
1. Set the value of `RUNNER_VERSION` to the version of the `model-runner` package that you want this control plane to use when running simulations.
   - The available package versions can be found [here](https://github.com/covid-policy-modelling/model-runner/packages/165741).
   - Note that package versions are taken from the last segment of branch names in the `model-runner` repo (e.g. `master` corresponds to the `master` branch, `0.3.0` corresponds to the tagged version `v0.3.0`, and `my-branch` corresponds to any branch named `some-prefix/my-branch`).
   - The `model-runner` repo is configured to automatically publish Docker images on updates to the different model packages it contains. For any additional models, you may need to set up similar automation, or manually publish their Docker images to a registry.

### API

The shared secret for the instance of the [web-ui](https://github.com/covid-policy-modelling/web-ui) that you want to coordinate with.

```shell script
API_SHARED_SECRET
```

### GitHub Packages

All of the Docker images that we have used to date are stored in [GitHub Packages](https://docs.github.com/en/free-pro-team@latest/packages/getting-started-with-github-container-registry/migrating-to-github-container-registry-for-docker-images) and required credentials with appropriate (read-only) permissions for all the images.
More about how to authenticate can be found in [the documentation](https://docs.github.com/en/free-pro-team@latest/packages/publishing-and-managing-packages/about-github-packages#authenticating-to-github-packages).
Note that the built-in Actions token (`secrets.GITHUB_TOKEN`) can access packages stored on this same repository; to access packages stored in other repositories, such as the `model-runner` repo, you may need to create a GitHub bot user with access to the repo and obtain a Personal Access Token for it.

It is possible to use additional/other Docker container registries.
If they require credentials, the appropriate login commands will need to be added to the "Log into registry" set of the "run" job.

```shell script
GPR_USER
GPR_PAT
```

### Azure

Azure storage credentials for storing the results of each model run.

```shell script
AZURE_STORAGE_ACCOUNT
AZURE_STORAGE_CONTAINER
```

### Artifacts

To enable artifact storage within GitHub Actions, set the following to a comma-separated list containing the types of artifact you want to retain.

```shell script
KEEP_ARTIFACTS=input,output,log
```

## Example: Multiple environments with promotion

We had three environments and corresponding control planes setup for testing new models and code:

* dev
* staging
* prod

Each environment will need to be its own distinct repo within your organization.

The majority of common changes (upgrading model and runner versions) can be done by simply change the corresponding Secret in each environment when you have deemed the change ready for promotion.

Any changes that we made to the workflow or control plane repo itself can be migrated between environments using a normal Git workflow.

### Pushing changes upstream

Check out the three control plane repositories.
Sibling directories are useful.
- dev
- staging
- prod

**Merging dev into staging**:

1. Navigate to your checkout of `dev-control-plane`.
1. Add a remote for `staging`:
   ```
   git remote add staging git@github.com:<your_org>/staging-control-plane.git
   ```
1. Update `master` and push to a branch on `staging`:
   ```
   git checkout master
   git pull origin master
   git push staging master:merge/dev-staging
   ```
1. Open a PR, get it approved, and merge it.

**Merging staging into prod**:

1. Navigate to your checkout of `staging-control-plane`.
1. Add a remote for prod:
   ```
   git remote add prod git@github.com:<your_org>/prod-control-plane.git
   ```
1. Update `master` and push to a branch on `prod`:
   ```
   git checkout master
   git pull origin master
   git push prod master:merge/staging-prod
   ```
1. Open a PR, get it approved and, merge it.
