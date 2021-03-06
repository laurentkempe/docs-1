---
type: docs
title: "Azure Key Vault with Managed Identities on Kubernetes"
linkTitle: "Azure Key Vault w/ Managed Identity"
description: How to configure Azure Key Vault and Kubernetes to use Azure Managed Identities to access secrets
---

## Component format

To setup Azure Key Vault secret store with Managed Identies create a component of type `secretstores.azure.keyvault`. See [this guide]({{< ref "secret-stores-overview.md#apply-the-configuration" >}}) on how to create and apply a secretstore configuration. See this guide on [referencing secrets]({{< ref component-secrets.md >}}) to retrieve and use the secret with Dapr components.

In Kubernetes mode, you store the certificate for the service principal into the Kubernetes Secret Store and then enable Azure Key Vault secret store with this certificate in Kubernetes secretstore.

The component yaml uses the name of your key vault and the Cliend ID of the managed identity to setup the secret store.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault
  namespace: default
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
  - name: vaultName
    value: [your_keyvault_name]
  - name: spnClientId
    value: [your_managed_identity_client_id]
```

{{% alert title="Warning" color="warning" %}}
The above example uses secrets as plain strings. It is recommended to use a local secret store such as [Kubernetes secret store]({{< ref kubernetes-secret-store.md >}}) or a [local file]({{< ref file-secret-store.md >}}) to bootstrap secure key storage.
{{% /alert %}}

## Spec metadata fields

| Field              | Required | Details                                                                 | Example             |
|--------------------|:--------:|-------------------------------------------------------------------------|---------------------|
| vaultName          | Y        | The name of the Azure Key Vault                                         | `"mykeyvault"`      |
| spnClientId        | Y        | Your managed identity client Id                                         | `"yourId"`          |

## Setup Managed Identity and Azure Key Vault

### Prerequisites

- [Azure Subscription](https://azure.microsoft.com/en-us/free/)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

### Steps

1. Login to Azure and set the default subscription

    ```bash
    # Log in Azure
    az login

    # Set your subscription to the default subscription
    az account set -s [your subscription id]
    ```

2. Create an Azure Key Vault in a region

    ```bash
    az keyvault create --location [region] --name [your keyvault] --resource-group [your resource group]
    ```

3. Create the managed identity(Optional)
   
    This step is required only if the AKS Cluster is provisoned without the flag "--enable-managed-identity". If the cluster is provisioned with manahed identity, than is suggested to use the autogenerated managed identity that is associated to the Resource Group MC_*.

    ```bash
    $identity = az identity create -g [your resource group] -n [you managed identity name] -o json | ConvertFrom-Json
    ```

    Below the command to retrieve the managed identity in the autogenerated scenario:

      ```bash
      az aks show -g <AKSResourceGroup> -n <AKSClusterName>
      ```
    For more detail about the roles to assign to integrate AKS with Azure Services [Role Assignment](https://azure.github.io/aad-pod-identity/docs/getting-started/role-assignment/).

4.  Retrieve Managed Identity ID
  
    The two main scenario are:
    - Service Principal, in this case the Resource Group is the one in which is deployed the AKS Service Cluster

    ```bash
    $clientId= az aks show -g <AKSResourceGroup> -n <AKSClusterName> --query servicePrincipalProfile.clientId -otsv
    ```

    - Managed Identity, in this case the Resource Group is the one in which is deployed the AKS Service Cluster

    ```bash
    $clientId= az aks show -g <AKSResourceGroup> -n <AKSClusterName> --query identityProfile.kubeletidentity.clientId -otsv
    ```
   
5. Assign the Reader role to the managed identity
  
    For AKS cluster, the cluster resource group refers to the resource group with a MC_ prefix, which contains all of the infrastructure resources associated with the cluster like VM/VMSS.

    ```bash
    az role assignment create --role "Reader" --assignee $clientId --scope /subscriptions/[your subscription id]/resourcegroups/[your resource group]
    ```

6. Assign the Managed Identity Operator role to the AKS Service Principal
  Refer to previous step about the Resource Group to use and which identity to assign
    ```bash
    az role assignment create  --role "Managed Identity Operator"  --assignee $clientId  --scope /subscriptions/[your subscription id]/resourcegroups/[your resource group]

    az role assignment create  --role "Virtual Machine Contributor"  --assignee $clientId  --scope /subscriptions/[your subscription id]/resourcegroups/[your resource group]
    ```

7. Add a policy to the Key Vault so the managed identity can read secrets

    ```bash
    az keyvault set-policy --name [your keyvault] --spn $clientId --secret-permissions get list
    ```

8. Enable AAD Pod Identity on AKS

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
    
    # For AKS clusters, deploy the MIC and AKS add-on exception by running -
    kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/mic-exception.yaml
    ```

9. Configure the Azure Identity and AzureIdentityBinding yaml

    Save the following yaml as azure-identity-config.yaml:

    ```yaml
    apiVersion: "aadpodidentity.k8s.io/v1"
    kind: AzureIdentity
    metadata:
      name: [you managed identity name]
    spec:
      type: 0
      resourceID: [you managed identity id]
      clientID: [you managed identity Client ID]
    ---
    apiVersion: "aadpodidentity.k8s.io/v1"
    kind: AzureIdentityBinding
    metadata:
      name: [you managed identity name]-identity-binding
    spec:
      azureIdentity: [you managed identity name]
      selector: [you managed identity selector]
    ```

10. Deploy the azure-identity-config.yaml:

    ```yaml
    kubectl apply -f azure-identity-config.yaml
    ```

## References
- [Azure CLI Keyvault CLI](https://docs.microsoft.com/en-us/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-create)
- [Create an Azure service principal with Azure CLI](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest)
- [AAD Pod Identity](https://github.com/Azure/aad-pod-identity)
- [Secrets building block]({{< ref secrets >}})
- [How-To: Retrieve a secret]({{< ref "howto-secrets.md" >}})
- [How-To: Reference secrets in Dapr components]({{< ref component-secrets.md >}})
- [Secrets API reference]({{< ref secrets_api.md >}})
