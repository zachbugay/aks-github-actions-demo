# Building and Deploying a C# Application to Azure Kubernetes Service with GitHub Actions

## Introduction

In this comprehensive guide, we'll walk through the complete process of setting up a modern CI/CD pipeline for a C# application using Azure Kubernetes Service (AKS), Azure Container Registry (ACR), and GitHub Actions. We'll leverage Azure CLI to provision all necessary infrastructure, implement proper security practices with managed identities and service principals, and automate the build and deployment process.

By the end of this article, you'll have a fully functional pipeline that automatically builds your C# application into a Docker container and deploys it to AKS whenever you push code changes to GitHub.

## Prerequisites

Before we begin, ensure you have the following:

- An active Azure subscription
- Azure CLI installed and configured (`az --version` to verify)
- A GitHub account
- Docker installed locally (for testing)
- .NET SDK installed (for the sample C# application)
- `kubectl` installed for Kubernetes management

## Architecture Overview

Our solution will include:

1. **Azure Container Registry (ACR)**: Private container registry for storing Docker images
2. **Azure Kubernetes Service (AKS)**: Managed Kubernetes cluster for running containers
3. **Managed Identity**: For AKS to pull images from ACR securely
4. **Service Principal**: For GitHub Actions to authenticate with Azure
5. **GitHub Actions**: CI/CD pipeline for automated builds and deployments

## Step 1: Set Up Environment Variables

First, let's define our environment variables to make the setup process easier and more consistent:

```PowerShell
# Define your variables
$RESOURCE_GROUP="rg-aks-github-demo"
$LOCATION="westus3"
$ACR_NAME="acraksdemo20251112"  # Must be globally unique
$AKS_CLUSTER_NAME="aks-github-demo-cluster"
$SERVICE_PRINCIPAL_NAME="sp-aks-github-actions"
$SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Display variables for verification
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "ACR Name: $ACR_NAME"
echo "AKS Cluster: $AKS_CLUSTER_NAME"
echo "Subscription ID: $SUBSCRIPTION_ID"
```

## Step 2: Create Resource Group

Create a resource group to contain all our Azure resources:

```PowerShell
az group create `
  --name $RESOURCE_GROUP `
  --location $LOCATION

echo "✓ Resource group created successfully"
```

## Step 3: Create Azure Container Registry (ACR)

Now let's create our Azure Container Registry where we'll store our Docker images:

```PowerShell
az acr create `
  --resource-group $RESOURCE_GROUP `
  --name $ACR_NAME `
  --sku Basic `
  --location $LOCATION

echo "✓ ACR created successfully"

# Get ACR login server
$ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --resource-group $RESOURCE_GROUP --query loginServer -o tsv)

echo "ACR Login Server: $ACR_LOGIN_SERVER"
```

> **Note**: We're using the Basic SKU for this demo. For production workloads, consider using Standard or Premium SKUs for better performance and features like geo-replication.

## Step 4: Create Azure Kubernetes Service (AKS) Cluster

Create an AKS cluster with managed identity enabled:

```PowerShell
az aks create `
  --resource-group $RESOURCE_GROUP `
  --name $AKS_CLUSTER_NAME `
  --node-count 2 `
  --node-vm-size Standard_D2lds_v6 `
  --enable-managed-identity `
  --generate-ssh-keys `
  --location $LOCATION `
  --attach-acr $ACR_NAME

echo "✓ AKS cluster created successfully"
```

This command does several important things:
- Creates a 2-node cluster with Standard_D2lds_v6 VMs
- Enables managed identity (recommended over service principal for cluster identity)
- Automatically configures AKS to pull images from our ACR using `--attach-acr`
- Generates SSH keys for node access

> **Best Practice**: Using `--attach-acr` automatically grants the AKS cluster's managed identity the `AcrPull` role on the specified ACR, eliminating the need for image pull secrets.

## Step 5: Get AKS Credentials

Configure `kubectl` to connect to your AKS cluster:

```PowerShell
az aks get-credentials `
  --resource-group $RESOURCE_GROUP `
  --name $AKS_CLUSTER_NAME

# Verify connection
kubectl get nodes

echo "✓ AKS credentials configured"
```

## Step 6: Create Service Principal for GitHub Actions

For GitHub Actions to authenticate with Azure, we'll create a service principal with appropriate permissions:

```PowerShell
# Create service principal and capture output
$SP_OUTPUT=$(az ad sp create-for-rbac `
  --name $SERVICE_PRINCIPAL_NAME `
  --role Contributor `
  --scopes /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP `
  --sdk-auth)

echo "✓ Service Principal created"
echo "⚠️  Save this output securely - you'll need it for GitHub Actions:"
echo "$SP_OUTPUT"

# Extract service principal details
$SP_APP_ID=$(echo $SP_OUTPUT | jq -r '.clientId')
$SP_PASSWORD=$(echo $SP_OUTPUT | jq -r '.clientSecret')
$SP_TENANT_ID=$(echo $SP_OUTPUT | jq -r '.tenantId')
```

> **Security Note**: Store these credentials securely. We'll add them to GitHub Secrets in a later step.

## Step 7: Grant ACR Push Permissions to Service Principal

Allow the service principal to push images to ACR:

```PowerShell
# Get ACR resource ID
$ACR_ID=$(az acr show `
  --name $ACR_NAME `
  --resource-group $RESOURCE_GROUP `
  --query id -o tsv)

# Assign AcrPush role to service principal
az role assignment create `
  --assignee $SP_APP_ID `
  --role AcrPush `
  --scope $ACR_ID

echo "✓ Service Principal granted ACR push permissions"
```

## Step 8: Create the C# Application

Now let's create a simple C# web application:

```PowerShell
# Create a new ASP.NET Core web API
dotnet new webapi -n Bugay.TestWebApi
cd Bugay.TestWebApi
```

Update `Program.cs` to add a custom endpoint:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.MapGet("/", () => "Hello from AKS!")
    .WithName("GetRoot")
    .WithOpenApi();

app.MapGet("/health", () => new { status = "healthy", timestamp = DateTime.UtcNow })
    .WithName("HealthCheck")
    .WithOpenApi();

app.Run();
```

## Step 9: Create Dockerfile

Create a `Dockerfile` in the `Bugay.TestWebApi` directory:

```dockerfile
# Use the official .NET SDK image for building
FROM mcr.microsoft.com/dotnet/nightly/sdk:10.0-preview AS build
WORKDIR /app

# Copy csproj and restore dependencies
COPY *.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/nightly/aspnet:10.0-preview
WORKDIR /app
COPY --from=build /app/out .

# Expose port
EXPOSE 5028
ENV ASPNETCORE_URLS=http://+:5028

ENTRYPOINT ["dotnet", "Bugay.TestWebApi.dll"]
```

## Step 10: Create Kubernetes Manifests

Create a directory for Kubernetes manifests:

```PowerShell
mkdir k8s
```

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: Bugay.TestWebApi
  labels:
    app: Bugay.TestWebApi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: Bugay.TestWebApi
  template:
    metadata:
      labels:
        app: Bugay.TestWebApi
    spec:
      containers:
      - name: Bugay.TestWebApi
        image: ${ACR_LOGIN_SERVER}/Bugay.TestWebApi:${IMAGE_TAG}
        ports:
        - containerPort: 5028
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5028
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 5028
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: Bugay.TestWebApi
spec:
  type: LoadBalancer
  selector:
    app: mywebapi
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5028
```

## Step 11: Create GitHub Actions Workflow

Create `.github/workflows/build-deploy.yml`:

```yaml
name: Build and Deploy to AKS

on:
  push:
    branches:
      - develop
  workflow_dispatch:

env:
  ACR_NAME: ${{ vars.ACR_NAME }}
  ACR_LOGIN_SERVER: ${{ vars.ACR_LOGIN_SERVER }}
  AKS_CLUSTER_NAME: ${{ vars.AKS_CLUSTER_NAME }}
  AKS_RESOURCE_GROUP: ${{ vars.RESOURCE_GROUP }}
  IMAGE_NAME: ${{ vars.IMAGE_NAME }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Build and push image to ACR
      run: |
        az acr build \
          --registry ${{ env.ACR_NAME }} \
          --image ${{ env.IMAGE_NAME }}:${{ github.sha }} \
          --image ${{ env.IMAGE_NAME }}:latest \
          --file MyWebApi/Dockerfile \
          ./MyWebApi
    
    - name: Set AKS context
      uses: azure/aks-set-context@v3
      with:
        resource-group: ${{ env.AKS_RESOURCE_GROUP }}
        cluster-name: ${{ env.AKS_CLUSTER_NAME }}
    
    - name: Deploy to AKS
      run: |
        # Replace placeholders in deployment file
        sed -i "s|\${ACR_LOGIN_SERVER}|${{ env.ACR_LOGIN_SERVER }}|g" k8s/deployment.yaml
        sed -i "s|\${IMAGE_TAG}|${{ github.sha }}|g" k8s/deployment.yaml
        
        # Apply Kubernetes manifests
        kubectl apply -f k8s/deployment.yaml
        
        # Wait for rollout to complete
        kubectl rollout status deployment/mywebapi
        
        # Get service external IP
        echo "Waiting for external IP..."
        kubectl get service mywebapi-service -w
```

## Step 12: Configure GitHub Repository

1. **Initialize Git repository**:

```PowerShell
cd aks-csharp-demo
git init
git add .
git commit -m "Initial commit: AKS C# demo application"
```

2. **Create GitHub repository** (using GitHub CLI or web interface):

```PowerShell
# If using GitHub CLI
gh repo create aks-csharp-demo --public --source=. --remote=origin --push
```

3. **Add Azure credentials to GitHub Secrets**:

   - Go to your GitHub repository
   - Navigate to **Settings** > **Secrets and variables** > **Actions**
   - Click **New repository secret**
   - Name: `AZURE_CREDENTIALS`
   - Value: Paste the entire JSON output from the service principal creation step

4. **Update the workflow file** with your actual ACR name:

```PowerShell
# Update the ACR_NAME and ACR_LOGIN_SERVER in .github/workflows/build-deploy.yml
sed -i "s/<YOUR_ACR_NAME>/$ACR_NAME/g" .github/workflows/build-deploy.yml
```

## Step 13: Test the Pipeline

Push your code to trigger the GitHub Actions workflow:

```PowerShell
git add .
git commit -m "Configure GitHub Actions workflow"
git push origin main
```

Monitor the workflow execution:
- Go to your GitHub repository
- Click on the **Actions** tab
- Watch your workflow run in real-time

## Step 14: Verify Deployment

Once the workflow completes, verify your deployment:

```PowerShell
# Check pods
kubectl get pods

# Check service
kubectl get service mywebapi-service

# Get external IP (may take a few minutes)
EXTERNAL_IP=$(kubectl get service mywebapi-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Application URL: http://$EXTERNAL_IP"

# Test the application
curl http://$EXTERNAL_IP
curl http://$EXTERNAL_IP/health
```

## Step 15: Monitoring and Troubleshooting

### View Application Logs

```PowerShell
# Get logs from all pods
kubectl logs -l app=mywebapi --tail=100

# Follow logs in real-time
kubectl logs -l app=mywebapi -f
```

### Describe Pods for Issues

```PowerShell
kubectl describe pod -l app=mywebapi
```

### Check Deployment Status

```PowerShell
kubectl get deployment mywebapi
kubectl rollout history deployment/mywebapi
```

## Best Practices Implemented

1. **Managed Identity**: Using managed identity for AKS-to-ACR authentication eliminates the need for storing credentials
2. **Resource Limits**: Defined CPU and memory requests/limits for predictable performance
3. **Health Checks**: Implemented liveness and readiness probes for automatic health monitoring
4. **Multi-stage Docker Build**: Optimized Docker image size using multi-stage builds
5. **Secrets Management**: Stored sensitive credentials in GitHub Secrets
6. **Image Tagging**: Used Git SHA for image tags to enable easy rollbacks
7. **Rolling Updates**: Kubernetes handles zero-downtime deployments automatically

## Security Considerations

- **Service Principal Scope**: Limited to specific resource group
- **RBAC**: Used minimum required permissions (Contributor on resource group, AcrPush on ACR)
- **No Hardcoded Secrets**: All secrets managed through GitHub Secrets or Azure managed identities
- **Private Registry**: ACR provides private container storage
- **Network Security**: Consider implementing Azure Network Policies for pod-to-pod communication

## Cost Optimization Tips

1. **Right-size your nodes**: Start with smaller VM sizes and scale as needed
2. **Use AKS cluster autoscaler**: Automatically adjust node count based on demand
3. **Implement pod autoscaling**: Use Horizontal Pod Autoscaler (HPA) for application scaling
4. **Stop dev/test clusters**: Use `az aks stop` to deallocate nodes when not in use
5. **Monitor ACR storage**: Implement retention policies to remove old images

## Cleanup Resources

When you're done, clean up all resources to avoid unnecessary charges:

```PowerShell
# Delete the resource group (this removes all resources within it)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

# Delete service principal
az ad sp delete --id $SP_APP_ID

echo "✓ Cleanup initiated"
```

## Conclusion

In this article, we've built a complete CI/CD pipeline for a C# application using Azure Kubernetes Service, Azure Container Registry, and GitHub Actions. We've implemented security best practices with managed identities and service principals, automated the build and deployment process, and created a scalable, production-ready infrastructure.

This setup provides a solid foundation for deploying containerized applications to Azure. You can extend this further by:

- Implementing Azure Application Insights for monitoring
- Adding multiple environments (dev, staging, production)
- Implementing GitOps with Flux or Argo CD
- Adding Azure Key Vault integration for secrets management
- Implementing advanced networking with Azure CNI or Calico

## Additional Resources

- [AKS Documentation](https://learn.microsoft.com/azure/aks/)
- [Azure Container Registry Documentation](https://learn.microsoft.com/azure/container-registry/)
- [GitHub Actions Documentation](https://docs.github.com/actions)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Azure CLI Reference](https://learn.microsoft.com/cli/azure/)

## About the Author

*This article demonstrates modern DevOps practices for deploying .NET applications to Azure Kubernetes Service. Feel free to adapt these patterns to your specific requirements and organizational standards.*

---

**Tags**: #Azure #AKS #Kubernetes #ACR #GitHub Actions #DevOps #CI/CD #CSharp #DotNet #Containers

