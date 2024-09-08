# Příprava prostředí - internal

![Schema architektury](./01b-Architektuira.png)

- ACA integrace s VNET
- KeyVault propojen s VNET přes Private Endpoint


## Nastaveni subskripce a az cli
```
az extension add --name containerapp --upgrade
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.KeyVault
```

Nastaveni proměnných pro dalsi snipety

```
LOCATION=swedencentral
RESOURCE_GROUP=ms-hackaton
VNET=vnet-1
KEYVAULT_NAME=hackaton-kv-42
ACR_NAME=hackatonregistry
ACA_ENV_NAME=ContainerAppEnv
ACA_APP_NAME=app-1
ACA_IDENTITY_NAME=ContainerAppEnv-uaa
```

## Create resource group
```
az group create \
    --location $LOCATION \
    --resource-group $RESOURCE_GROUP
```


## Create vnet

```
az network vnet create \
    --name $VNET \
    --resource-group $RESOURCE_GROUP \
    --address-prefix 10.0.0.0/16

az network vnet subnet create \
    --name ACA-subnet \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET \
    --address-prefix 10.0.0.0/24
```

### Crete private endpoint subnet
```
az network vnet subnet create \
    --name PrivateEndpoint-subnet \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET \
    --address-prefix 10.0.1.0/25
```

### Crete bastion subnet
```
az network vnet subnet create \
    --name AzureBastion-subnet \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET \
    --address-prefix 10.0.1.128/25
```


## Private DNS


### Create a Private DNS Zone
```
az network private-dns zone create \
    --resource-group $RESOURCE_GROUP \
    --name privatelink.vaultcore.azure.net
```

### Link the Private DNS Zone to the Virtual Network
```
az network private-dns link vnet create \
    --resource-group $RESOURCE_GROUP \
    --virtual-network $VNET \
    --zone-name privatelink.vaultcore.azure.net \
    --name vnet-1-to-vaultcore-dns-link \
    --registration-enabled true
```


## Bastion

### Create public ip for bastion
```
az network public-ip create \
    --resource-group $RESOURCE_GROUP \
    --name bastion-publicIp \
    --sku Standard \
    --zone 1 2 3
```

### Create bastion
```
az network bastion create \
    --name bastion \
    --public-ip-address bastion-publicIp \
    --resource-group $RESOURCE_GROUP \
    --vnet-name vnet-1 \
```


## Container Registry

```
az acr create \
    --resource-group $RESOURCE_GROUP \
    --name $ACR_NAME \
    --sku Basic

ACR_ID=`az acr show --resource-group $RESOURCE_GROUP --name $ACR_NAME --query "id" -o tsv`
```


## KeyVault#

```
az keyvault create \
    --name $KEYVAULT_NAME \
    --resource-group $RESOURCE_GROUP
```

### Turn on Key Vault Firewall
```
az keyvault update \
    --name $KEYVAULT_NAME \
    --resource-group $RESOURCE_GROUP \
    --default-action deny
```

### Disable Virtual Network Policies
```
az network vnet subnet update \
    --name PrivateEndpoint-subnet \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET \
    --private-endpoint-network-policies Disabled
```

```
az network private-endpoint create \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET \
    --subnet PrivateEndpoint-subnet \
    --name hackaton-keyvault-pe \
    --private-connection-resource-id "/subscriptions/b36a9917-4893-4f01-9b34-cf8ad8b15df2/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KEYVAULT_NAME" \
    --group-ids vault \
    --connection-name hackaton-keyvault-pe
```

### Determine the Private Endpoint IP address
```
PE_NIC_ID=`az network private-endpoint show --resource-group $RESOURCE_GROUP -n hackaton-keyvault-pe --query "networkInterfaces[0].id" -o tsv`
PE_NIC_PRIVATE_IP=`az network nic show --ids $PE_NIC_ID --query "ipConfigurations[0].privateIPAddress" -o tsv`

az network private-dns record-set a add-record \
    --resource-group $RESOURCE_GROUP \
    -z "privatelink.vaultcore.azure.net" \
    -n $KEYVAULT_NAME \
    -a $PE_NIC_PRIVATE_IP
```


## Contariner App Environment

```
az network vnet subnet update \
    --resource-group $RESOURCE_GROUP \
    --name ACA-subnet \
    --vnet-name $VNET \
    --delegations Microsoft.App/environments

INFRASTRUCTURE_SUBNET=`az network vnet subnet show --resource-group $RESOURCE_GROUP --vnet-name $VNET --name ACA-subnet --query "id" -o tsv | tr -d '[:space:]'`
```

```
az identity create \
    --resource-group $RESOURCE_GROUP \
    --name $ACA_IDENTITY_NAME

ACA_IDENTITY_ID=`az identity show --name $ACA_IDENTITY_NAME -g $RESOURCE_GROUP --query "id" -o tsv`
```

```
az role assignment create \
    --assignee $ACA_IDENTITY_ID \
    --role AcrPull \
    --scope $ACR_ID
```

## Create container app environment
```
az containerapp env create \
  --name $ACA_ENV_NAME \
  --resource-group $RESOURCE_GROUP \
  --infrastructure-subnet-resource-id $INFRASTRUCTURE_SUBNET \
  --location $LOCATION \
  --internal-only \
  --enable-workload-profiles
```

```
az containerapp create \
    --name $ACA_APP_NAME  \
    --resource-group $RESOURCE_GROUP \
    --environment $ACA_ENV_NAME \
    --user-assigned /subscriptions/XXXX/resourcegroups/$RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$ACA_IDENTITY_NAME \
    --ingress internal \
    --target-port 80
```