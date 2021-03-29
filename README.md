# Umbrella Helm Chart for deploying apps to AKS 

This helm chart can be used to easily deploy apps to your AKS cluster configured with Pod Identity, an Application Gateway and an Azure Key Vault where pods can directly read secrets from. 

This chart is heavily dependent on these resources and builds upon the [Fully Configured AKS Terraform Deployment repostitory](https://github.com/bart-jansen/terraform-aks-appgw-acr-keyvault-loganalytics) that deploys these to Azure.

## Prequisites:
- [helm v3](https://helm.sh/docs/intro/install/)
- [azure-cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Fully configured AKS deployement](https://github.com/bart-jansen/terraform-aks-appgw-acr-keyvault-loganalytics)

## Overview

This repo provides two helm charts:
- `/helm-app/` - umbrella helm chart available on Docker Hub
- `/helm-app-example/` - app helm chart built on the umbrella helm chart above (as a Chart dependency) 

The umbrella `helm-app` Chart provides these templates:
- `deployment.yaml` - AKS deployment with the Container Pod
- `ingress.yaml` - Ingress with all redirect rule (automatically applied to AppGateway)
- `secretproviderclass.yaml` - SecretProviderClass that fetches secrets from Azure Key Vault and exposes them to other pods through podidentity
- `service.yaml` - The service that hosts the app `deployment.yaml`


## Configuration

An example of all configuration parameters and its default values is shown in (./helm-app/values.yaml)[helm-app/values.yaml]. When you use this umbrella helm chart as a dependency, you only have to define the parameters you want to change. Leaving parameters out will automatically set them to their default values.

### Config - App

| Name | Description | Default |
|------|-------------|---------|
| `name` | App name | app-service-name  | 
| `aadpodidbinding` | Name of your Azure AD Pod Identity binding<sup>1</sup> | podidentity | 
| `replicaCount` | Amount of replicas you want for your pod | 1 | 
| `secretEnv (list)` | List of Environment Variables fetched from Azure Key Vault | [] | 
| `- envName` | Environment variable name |  | 
| `- secretKey` | Secret name from Azure Key Vault |  | 
| `env (list)` | Kubernetes version of the node pool | [] | 
| `- key` | Environment variable name |  | 
| `- value` | Value of environment variable | |
| `container (object)` | Container object | {} | 
| `- image` | Container image that runs inside the pod | ghcr.io/your-username/your-repo:your-tag | 
| `- port` | Container port the image is hosted on | 80 | 
| `- healthCheckHttpGetPath` | Path for Application Gateway to check for a healthy endpoint<sup>2</sup> | / | 


> <sup>1</sup> Equals podidentity binding if you didn't change the default values in the terraform deployment<br/>
 <sup>2</sup> Healthy HTTP StatusCode required, between 200 and 399


### Config - Ingress

| Name | Description | Default |
|------|-------------|---------|
| `name` | Name of your ingress | app-ingress-name  | 
| `replicaCount` | Amount of replicas you want for your ingress | 1 | 
| `dnsName` | DNS Name of your Application Gateway Public IP (without https()) |  | 
| `paths (list)` | List of ingress paths | [] | 
| `- path` | Path where pod is hosted | /* | 
| `- backend: serviceName` | Name of the service defined in app | app-service-name |
| `- backend: servicePort` | Port where container is hosted | 80 |


### Config - SecretStore

| Name | Description | Default |
|------|-------------|---------|
| `provider` | Name of your SecretStore provider | azure  | 
| `usePodIdentity` | Boolean to trigger use of Pod Identity | true | 
| `keyvaultName` | Name of the Azure Key Vault to fetch secrets from | yourkeyvaultname | 
| `secrets (array)` | List of secrets to fetch (e.g. ["secret1", "secret2"]) | [] | 
| `tenantId` | TenantID of your Azure tenant | 00000000-0000-0000-0000-000000000000 | 

## Usage

1. Clone repo

```
git clone https://github.com/bart-jansen/secretstore-ingress-apps-helm.git
```

2. `cd` into  `/helm-app-example` and make appropriate changes to `Chart.yaml` and `values.yaml`

3. Login to your AKS cluster (if you haven't already):

```
az aks get-credentials -n youraksclustername -g yourresourcegroupname
```

4. Install Helm Chart:

```
helm upgrade --install --namespace aksnamespace -f values.yaml your-app-name .
```
