# Autoscaling

## Cíl
- otestovat škálování na základě:
    - CPU zátěže
    - počtu spojení
    - pomoci KEDA na základě počtu objektů ve frontě


# Materialy a postup


https://learn.microsoft.com/en-us/azure/container-apps/scale-app?pivots=azure-resource-manager


```
LOCATION=swedencentral
RESOURCE_GROUP=ms-hackaton-dodo-public
KEYVAULT_NAME=hackaton-dodo-kv-43
ACR_NAME=hcdodoacr43
ACA_ENV_NAME=dodo-ContainerAppEnvPublic
ACA_APP_NAME=dodo-app-1
ACA_IDENTITY_NAME=dodo-ContainerAppEnv-uaa
```

## Connection

```
az containerapp update \
	--name $ACA_APP_NAME \
	--resource-group $RESOURCE_GROUP \
    --scale-rule-name my-http-scale-rule \
    --scale-rule-http-concurrency 1
```

## Keda

https://learn.microsoft.com/en-us/azure/container-apps/scale-app?pivots=azure-resource-manager#custom
https://keda.sh/docs/2.15/scalers/azure-storage-queue/