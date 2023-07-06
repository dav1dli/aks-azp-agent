# Azure Pipelines Agent container
[Azure Pipeline Agent](https://github.com/microsoft/azure-pipelines-agent/tree/master) is [required](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser) to run pipelines in [Azure DevOps Services](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser).

This repository containes the code building a containerized agent which can be deployed to AKS.

Agent container build is automated with AZDO pipilines in `devops/pipelines`.

The agent expects following environment variables:
* AZP_URL - AZDO organization URL https://dev.azure.com/{your-organization} (for example, https://dev.azure.com/Organization)
* AZP_TOKEN - [PAT token](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops#authenticate-with-a-personal-access-token-pat)
* AZP_POOL - [AZDO agent pool](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops&tabs=yaml%2Cbrowser)

See the [agent documentation](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser) and specifically [Linux self-hosted agents](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops) for more details.

## Pre-requirements
* Azure DevOps organization
* Agent pool
* PAT token
* ACR container registry
* az cli

For agent scaling KEDA is required on AKS.

## Build
The container is built using provided `Dockerfile`. The build follows [official instructions](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#linux) but installs the agent under unprivileged user. The build is automated with `devops/pipelines/img-build.yaml` pipeline. Pipelines are configured in `devops/pipelines/config/{ENVIRONMENT}.yaml` files corresponding to environments.

The container image is pushed to ACR registry using a unique sequential and latest tag.

Helm chart provides a deployment configuration, is built and published to ACR by the pipeline.

## Run
### Azure Kubernetes Service
See `doc/aks.md` for details how to deploy and run the agent on AKS.

## Test
Test pipeline `devops/pipelines/agent-test.yaml` runs on the configured agent and tests its capabilities.

### Use cases

#### Azure Cloud
The agent image is pre–installed with `az` cli. Authentication comes from Service Connection configured in Azure DevOps. Task `AzureCLI@2` provides context for running `az` commands.

#### Kubernetes: kubectl and helm
The test pipeline shows how to perform actions on a Kubernetes/AKS cluster from a pipeline. Azure pipeline provides tasks for installing kubernetes tools. Command `az aks get-credentials` configures kubeconfig, variable `KUBECONFIG` is supported by both `kubectl` and `helm`.

#### Container images builds
The image DOES NOT include any tools for building container images. Instead, use methods like kaniko build. It starts a pod, pass it the build context (sources, Dockerfile, parameter where to push the built image, image tags) and pushes an image to a container registry. Follow its lifecycle to see what's the build status.

#### Terraform
Terraform can be supported similarly to k8s commands use case: Azure pipelines provides `TerraformInstaller@0` task. Terraform Azure provider respects following variables:
```
ARM_SUBSCRIPTION_ID=$(az account show --query "id" -otsv)
ARM_TENANT_ID=$(az account show --query "tenantId" -otsv)
ARM_CLIENT_ID=$(az ad app list --display-name AZU-XXX-POC-DL --query "[0].appId" -otsv)
ARM_CLIENT_SECRET - when created must be saved securely, for example in a key vault
```
Terrafrom can run from a context of `AzureCLI@2` task or [Azure CLI](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/azure_cli) session, then it does not need those variables set explicitly.
