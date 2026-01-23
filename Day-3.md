
## Day 3 – Azure AKS Setup

**AKS architecture (high level)**  
- AKS is a managed Kubernetes service: Azure runs the control plane, you manage the worker nodes and workloads.  
- A cluster is split into:  
  - Control plane (API server, etcd, scheduler, controllers) – managed by Azure.  
  - Node pools – sets of VMs that run your Pods (system and user node pools).  
- Common setup: AKS in a virtual network, exposed via an Azure Load Balancer or Application Gateway, integrated with monitoring and other Azure services.

**Azure Container Registry (ACR)**  
- Private container registry in Azure, similar to Docker Hub but scoped to your subscription.  
- Each registry has a login server like:  
  - `myregistry.azurecr.io`  
- You log in via Azure CLI, then push/pull images using standard Docker commands.

**Pushing images to ACR – flow**  
1. Create ACR.  
2. Build your Docker image locally.  
3. Tag the image with the ACR login server.  
4. Push the image to ACR.

***

## ACR – Commands and Flow

Set some variables:

```bash
RESOURCE_GROUP=my-rg
ACR_NAME=myacr12345      # must be globally unique
LOCATION=eastus
IMAGE_NAME=fastapi-demo
IMAGE_TAG=v1
```

**1. Create ACR**

```bash
az group create -n $RESOURCE_GROUP -l $LOCATION

az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Basic
```

**2. Log in to ACR**

```bash
az acr login --name $ACR_NAME
```

**3. Build and tag image**

Build:

```bash
docker build -t $IMAGE_NAME:$IMAGE_TAG .
```

Get the login server:

```bash
ACR_LOGIN_SERVER=$(az acr show \
  --name $ACR_NAME \
  --query loginServer \
  --output tsv)
```

Tag for ACR:

```bash
docker tag $IMAGE_NAME:$IMAGE_TAG \
  $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
```

**4. Push to ACR**

```bash
docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
```

(Optional) List repositories:

```bash
az acr repository list --name $ACR_NAME --output table
```

***

## Create AKS Cluster and Attach ACR

### 1. Create AKS cluster

```bash
AKS_NAME=my-aks-cluster

az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --node-count 2 \
  --generate-ssh-keys
```

Get kubeconfig:

```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME
```

Check nodes:

```bash
kubectl get nodes
```

### 2. Give AKS access to ACR

Attach ACR to AKS:

```bash
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --attach-acr $ACR_NAME
```

This lets Pods pull images from ACR without extra imagePullSecrets.

***

## Deploy API to AKS (Using ACR Image)

Assume your image is:

```text
$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
# e.g. myacr12345.azurecr.io/fastapi-demo:v1
```

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-aks-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fastapi-aks
  template:
    metadata:
      labels:
        app: fastapi-aks
    spec:
      containers:
      - name: fastapi-aks
        image: myacr12345.azurecr.io/fastapi-demo:v1  # change to your ACR + image
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: "500m"
            memory: "256Mi"
```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-aks-service
spec:
  type: LoadBalancer
  selector:
    app: fastapi-aks
  ports:
  - port: 80
    targetPort: 80
```

Apply manifests:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Check resources:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc fastapi-aks-service
```

Look for an `EXTERNAL-IP` on the service (may take a minute).

***

## Deliverable: API Reachable via AKS Service

Validation checklist:

- Deployment has desired number of Pods in `Running` state.  
- Service type is `LoadBalancer` and has a public `EXTERNAL-IP`.  
- Hitting `http://<EXTERNAL-IP>` returns your API response.  
- Hitting `http://<EXTERNAL-IP>/docs` (for FastAPI) shows the API docs.

