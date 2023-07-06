# Azure Pipelines Agent on Azure Kubernetes Service

Generate KUBECONFIG credentials for a selected cluster:

```bash
az account set --subscription XXXXX-YYYY-ZZZZ-b357-ABCDEFGH
az aks get-credentials --resource-group RGTEST1 --name AKSTEST1
```

Helm registry login: `helm registry login -u $(ACRUSER) -p $(ACRPASS) $(ACR)`

Deploy the agent:
```
helm install $(imgname) oci://$(ACR)/helm/azp-agent-job \
  --set imagePullSecret.username={USER} \
  --set imagePullSecret.password={PASSWORD} \
  --set azp.token=XXXX \
  -f helm/azp-agent-job/values.poc.yaml \
  --namespace azp --create-namespace --atomic --debug
```
*Note:* for the scaling up an agent as a job one agent in an agent pool is required, even if not running. That's why the chart creates a job template.

## Security
Since the agent runs workloads on Kubernetes and thus accesses Kubernetes API it is required that it mounts API token so it is not disabled in security restructions.

## Scaling
Use KEDA trigger [azure-pipelines](https://keda.sh/docs/2.10/scalers/azure-pipelines/) to scale the agent job up and down.

## Automation
Pipeline `devops/pipelines/aks-deploy.yaml` provides an automation of the agent deployment. The agent is provisioned with secret parameters taken from the environment context, such as a key vault.

*Note:* for the pipeline to run an agent is required. In case of private AKS clusters the agent must have access to cluster's private endpoint, for example run in the same VNET. Deploying the chart manually allows to run the pipeline. Then the manually deployed agent can be removed both from the cluster and from Azure DevOps agent pool.