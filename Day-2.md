
## Day 2 – Kubernetes Basics


**Pods**  
- Smallest unit you deploy.  
- One Pod = 1+ tightly coupled containers sharing network/storage (like siblings).  
- Pods are ephemeral (they die, get replaced). Don’t manage them directly.  

**Deployments**  
- Manages ReplicaSets, which manage multiple identical Pods.  
- Handles scaling, updates, rollouts, rollbacks.  
- Example: “Keep 3 copies of my app running; if one crashes, replace it.”  

**Services**  
- Provides stable IP/DNS for a set of Pods.  
- Load balances traffic across matching Pods (by label).  
- Types: ClusterIP (internal), NodePort/LoadBalancer (external access).  

**kubectl**  
- CLI to talk to the cluster.  
- Basic commands:  
  - `kubectl get pods/deployments/services` – list resources.  
  - `kubectl apply -f deployment.yaml` – create/update from YAML.  
  - `kubectl describe pod <name>` – detailed info.  
  - `kubectl logs <pod-name>` – view container logs.  
  - `kubectl port-forward svc/<service> 8000:80` – access service locally.  

**YAML manifests**  
- Declarative files defining resources (kind: Deployment, Service).  
- Applied with `kubectl apply -f file.yaml`.  

***

### Local Kubernetes Setup

Use **Minikube** (single‑node cluster, easy for local dev):  

```bash
# Install (if needed): brew install minikube kubectl (on Mac) or equivalent
minikube start
kubectl get nodes  # should show your minikube node
```

**Alternative: kind** (Kubernetes IN Docker, lighter):  
```bash
kind create cluster
```

Both work for deploying your Dockerized FastAPI API.  

***

## Build: Deploy FastAPI to Local K8s

1. **Prep your image**  
   - Build/push your FastAPI Docker image to a registry (or use Minikube’s Docker env).  
   - For Minikube:  
     ```bash
     eval $(minikube docker-env)
     docker build -t fastapi-demo .
     ```  

2. **Create YAML files** (deliverables below).  

3. **Deploy**  
   ```bash
   kubectl apply -f deployment.yaml
   kubectl apply -f service.yaml
   
   kubectl get pods  # wait for Running
   kubectl get svc   # note the service details
   ```

4. **Access the API**  
   ```bash
   minikube service fastapi-service  # opens browser
   # or
   kubectl port-forward svc/fastapi-service 8000:80
   # then http://localhost:8000
   ```  

***

## Deliverables

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
spec:
  replicas: 2  # 2 pod copies
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
      - name: fastapi
        image: fastapi-demo:latest  # your Docker image
        ports:
        - containerPort: 80  # what your FastAPI listens on
        resources:
          limits:
            cpu: "500m"
            memory: "256Mi"
```

**What this does**: Creates 2 Pods running your image. Scales, restarts if needed.  

***

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  type: LoadBalancer  # or NodePort for Minikube
  selector:
    app: fastapi  # matches Deployment pods
  ports:
  - port: 80
    targetPort: 80
```

**What this does**: Stable endpoint for the Pods. Load balances traffic. Minikube assigns external IP/port.  

***

## Validation Commands

```bash
kubectl get deployments,pods,services
kubectl describe deployment fastapi-deployment
kubectl logs deployment/fastapi-deployment
```

Scale test: `kubectl scale deployment fastapi-deployment --replicas=3`  

Cleanup: `kubectl delete -f deployment.yaml -f service.yaml`  

***

These YAMLs deploy your Dockerized FastAPI from yesterday. Tweak image tag/port as needed. Next day can add ConfigMaps/Secrets or Ingress.

