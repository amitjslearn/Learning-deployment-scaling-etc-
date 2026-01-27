
**GitHub Actions / Azure DevOps**  
- GitHub Actions: YAML workflows in `.github/workflows/` that run on GitHub-hosted runners or self-hosted agents. Events like `push`, `pull_request` trigger jobs.  
- Azure DevOps Pipelines: Similar YAML-based pipelines in Azure DevOps, tightly integrated with Azure services like ACR/AKS.  
- Both support multi-stage pipelines: CI (build/test) → CD (deploy).  

**Build → Test → Deploy flow**  
- **CI stage**: Checkout code → run tests → build Docker image → push to registry (ACR).  
- **CD stage**: Update Kubernetes manifests (with new image tag) → apply to cluster.  
- Secrets (Azure credentials) stored in GitHub Secrets or Azure DevOps variables.  
- Use commit SHA or semantic versioning for image tags to ensure traceability.  

***

## GitHub Actions Pipeline for AKS (Recommended for GitHub repos)

Create `.github/workflows/ci-cd.yaml`:

```yaml
name: CI/CD to AKS

env:
  ACR_NAME: myacr12345
  AKS_RESOURCE_GROUP: my-rg
  AKS_CLUSTER: my-aks-cluster
  CONTAINER_NAME: fastapi-demo
  IMAGE_TAG: ${{ github.sha }}

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Login to ACR
      uses: azure/docker-login@v2
      with:
        login-server: ${{ env.ACR_NAME }}.azurecr.io
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
    
    - name: Build and push
      run: |
        docker build . -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.CONTAINER_NAME }}:${{ env.IMAGE_TAG }}
        docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.CONTAINER_NAME }}:${{ env.IMAGE_TAG }}
    
    # Optional: run tests here (pytest, etc.)

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v4
    
    - name: Azure login
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Set AKS context
      uses: azure/aks-set-context@v4
      with:
        resource-group: ${{ env.AKS_RESOURCE_GROUP }}
        cluster-name: ${{ env.AKS_CLUSTER }}
    
    - name: Deploy to AKS
      run: |
        sed -i 's|image: .*|image: ${{ env.ACR_NAME }}.azurecr.io/${{ env.CONTAINER_NAME }}:${{ env.IMAGE_TAG }}|g' deployment.yaml
        kubectl apply -f deployment.yaml -f service.yaml
```

**Setup secrets** (GitHub → Settings → Secrets and variables → Actions):

- `AZURE_CREDENTIALS`: JSON from Azure Service Principal (SP). Create SP: `az ad sp create-for-rbac --sdk-auth`.  
- `ACR_USERNAME` / `ACR_PASSWORD`: From `az acr credential show`.  

***

## Azure DevOps Pipeline (Alternative)

If using Azure DevOps:

1. Create pipeline → "Deploy to Azure Kubernetes Service".  
2. Stages:  
   - **Build**: `Docker@2` task → build/push to ACR.  
   - **Deploy**: `Kubernetes@1` task → `kubectl apply` manifests.  

YAML snippet for deploy stage:

```yaml
- task: Kubernetes@1
  inputs:
    connectionType: 'Azure Resource Manager'
    azureSubscriptionEndpoint: 'your-service-connection'
    azureResourceGroup: 'my-rg'
    kubernetesCluster: 'my-aks-cluster'
    command: 'apply'
    arguments: '-f deployment.yaml -f service.yaml'
    secretType: 'generic'
```

Service connection: Azure Resource Manager type linked to your subscription.

***

## Build: CI + CD Setup

1. **Prep repo**  
   - Add `deployment.yaml`, `service.yaml` to repo.  
   - Ensure Dockerfile exists.  

2. **Create secrets** (as above).  

3. **Push to trigger**  
   - `git add . && git commit -m "Add CI/CD" && git push origin main`.  

4. **Verify pipeline**  
   - GitHub: Actions tab → watch build/push/deploy jobs.  
   - Check AKS: `kubectl get pods/deployments` → new image tag deployed.  

***

## Deliverable: One-click Deployment via Git Push

**Test it**:  
- Make a small change (e.g. update FastAPI message).  
- `git commit -m "Update message" && git push origin main`.  
- Pipeline runs automatically → new image built/pushed → AKS updated.  
- API shows new version (hit `/docs` or endpoint).  

**Key indicators**:  
- Actions tab shows green checkmarks on build + deploy jobs.  
- `kubectl describe deployment fastapi-aks-deployment` shows new image digest/tag.  
- No manual kubectl/helm commands needed after initial setup.  

Pipeline ensures: tests pass → image built → deployed safely. Scale it by adding more stages (security scans, approvals).

