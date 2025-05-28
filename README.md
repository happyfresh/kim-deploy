
# Deploy to Azure Container App (ACA) Using YAML

## 1. Login to Azure

```bash
az login
```

> _Note:_  
> You can use azure-cli docker for easy installation  
> `docker run -it -v <LOCAL_DIR>:/mnt/current" mcr.microsoft.com/azure-cli`

## 2. Set the correct subscription

```bash
az account set --subscription <subscription-id>
```

## 3. Pull latest image from GitLab and push to Azure Container Registry (ACR)

```bash
# Login to ACR
az acr login --name <acr-name>

# Pull image from GitLab
docker pull gitlab/image-name:tag

# Tag image for ACR
docker tag gitlab/image-name:tag image-name:tag

# Push to ACR
docker push image-name:tag
```

## 4. Set environment variables

```bash
export RESOURCE_GROUP=""
export LOCATION="southeastasia"
export ACR_NAME=""
export CONTAINER_APP_NAME="kim-staging"
export IMAGE_NAME="${ACR_NAME}.azurecr.io/<image-name:tag>"
export CONTAINER_APP_ENV_NAME="your-env"
```

## 5. Create the Container App Environment (if not already created)

```bash
az containerapp env create \
  --name $CONTAINER_APP_ENV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION
```

## 6. Create initial minimal Container App using Hello World image

```bash
az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINER_APP_ENV_NAME \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --ingress external \
  --target-port 80 \
  --system-assigned
```

## 7. Create Secrets for the App

```bash
az containerapp secret set \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --secrets \
    database-url=postgres://xxx \
    dona-api-key=xxx \
    nextauth-secret=xxx
```

## 8. Grant ACR Pull Permission to Container App's Managed Identity

```bash
# Get the managed identity principal ID
PRINCIPAL_ID=$(az containerapp show \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query identity.principalId \
  --output tsv)

echo "Managed Identity Principal ID: $PRINCIPAL_ID"

# Get the ACR resource scope
ACR_SCOPE=$(az acr show --name $ACR_NAME --query id --output tsv)

echo "ACR resource scope: $ACR_SCOPE"

# Assign AcrPull role to container app's managed identity on the ACR scope
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "AcrPull" \
  --scope $ACR_SCOPE
```

## 9. Link Container App's Managed Identity to ACR Registry

```bash
az containerapp registry set \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --server "${ACR_NAME}.azurecr.io" \
  --identity system
```

> This step explicitly grants the container app's managed identity permission to authenticate with the Azure Container Registry.

## 10. Prepare the full Container App YAML (`containerapp.yaml`)

Save the following as `containerapp.yaml`:

```yaml
name: kim-staging-dry-run
type: microsoft.app/containerapps
location: southeastasia
identity:
  type: SystemAssigned
properties:
  environmentId: /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.App/managedEnvironments/<container-app-env-name>
  configuration:
    ingress:
      external: true
      targetPort: 3000
    secrets:
      - name: db-url
        value: xxx
      - name: dona-api-key
        value: xxx
      - name: nextauth-secret
        value: xxx
    activeRevisionsMode: Single
  template:
    containers:
      - name: kim-container
        image: <image-name:tag>
        resources:
          cpu: 1.0
          memory: 2Gi
        env:
          - name: APP_ENV
            value: production
          - name: DONA_HOST
            value: https://dona-api-staging.example.com
          - name: NEXTAUTH_URL
            value: http://example.com
          - name: NEXT_PUBLIC_SENTRY_DSN
            value: https://example.com
          - name: SENTRY_TRACES_SAMPLE_RATE
            value: "1.0"
          - name: DONA_API_KEY
            secretRef: dona-api-key
          - name: NEXTAUTH_SECRET
            secretRef: nextauth-secret
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
```

## 11. Update the Container App using YAML

```bash
az containerapp update \
  --resource-group $RESOURCE_GROUP \
  --name $CONTAINER_APP_NAME \
  --yaml containerapp.yaml
```
