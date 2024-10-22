# AKS with Redis Cache sample (doc WIP)

## Overview
How to connect to Azure Cache for Redis from an application running in AKS using an Microsoft Entra identity and K8S workload identity. This code sample can also be run locally with your user principal configured as a Redis User for your redis instance. Based on the [tutorial](https://docs.microsoft.com/azure/azure-cache-for-redis/cache-tutorial-aks-get-started).

## Instructions
1. Update the cluster to work with OIDC and workload identity

`az aks update --resource-group "rg-elias-aks" --name "eliasaks" --enable-oidc-issuer --enable-workload-identity`

2. Get the OIDC issuer

`az aks show --name "eliasaks" --resource-group "rg-elias-aks" --query "oidcIssuerProfile.issuerUrl" --output tsv`

3. Create a Kubernetes service account

`az aks get-credentials --name "eliasaks" --resource-group "rg-elias-aks"`

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: "7f34b5c0-2509-4c80-8acc-f19c1eea34b4"
  name: "workload-identity-sa"
  namespace: "default"
EOF
```

4. Create federated identity

`az identity federated-credential create --name "myFedIdentity" --identity-name "eliasmanagedidaks" --resource-group "rg-elias-aks" --issuer "https://eastus.oic.prod-aks.azure.com/b25036e3-de39-4fec-a4aa-bda41b870d38/4327d83a-dd12-4dc6-9a45-f303f763fa99/" --subject system:serviceaccount:"default":"workload-identity-sa" --audience api://AzureADTokenExchange`

5. Upload workload app
```bash
git clone https://github.com/elias-rod/azure-cache-redis-sample/
cd azure-cache-redis-sample/sample
az acr build --image sample/connect-from-aks-sample:1.0 --registry eliasaksregistry --file Dockerfile .
az aks update --name eliasaks --resource-group rg-elias-aks --attach-acr eliasaksregistry
```

6. Create podspec.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: entrademo-pod
  labels:
    azure.workload.identity/use: "true"  # Required. Only pods with this label can use workload identity.
spec:
  serviceAccountName: workload-identity-sa
  containers:
  - name: entrademo-container
    image: eliasaksregistry.azurecr.io/sample/connect-from-aks-sample:1.0
    imagePullPolicy: Always
    command: ["dotnet", "ConnectFromAKS.dll"] 
    resources:
      limits:
        memory: "256Mi"
        cpu: "500m"
      requests:
        memory: "128Mi"
        cpu: "250m"
    env:
         - name: AUTHENTICATION_TYPE
           value: "WORKLOAD_IDENTITY" # change to ACCESS_KEY to authenticate using access key
         - name: REDIS_HOSTNAME
           value: "redisCache-uzafjwwnzwzqc.redis.cache.windows.net"
         - name: REDIS_ACCESSKEY
           value: "eAeEetcetc" 
         - name: REDIS_PORT
           value: "6380"
  restartPolicy: Never
```
## Try
1. Create the pod
`kubectl apply -f podspec.yaml`

2. View the logs
`kubectl logs entrademo-pod`
