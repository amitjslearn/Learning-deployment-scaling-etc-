
## Day 4 – Ingress & Scaling

**Ingress controllers**  
- An Ingress is a Kubernetes object that defines HTTP/HTTPS routing (hostnames, paths) to Services inside the cluster.  
- An Ingress controller is the actual component (like NGINX Ingress) that watches Ingress resources and configures a reverse proxy/load balancer accordingly.  
- NGINX Ingress controller typically runs as a Deployment + Service (often `LoadBalancer` or `NodePort`) and becomes the main entry point to your cluster.  

**Horizontal Pod Autoscaler (HPA)**  
- HPA automatically changes the number of Pod replicas for a Deployment/ReplicaSet/StatefulSet based on metrics (commonly CPU).  
- You define min/max replicas and a target CPU utilization percentage; the controller scales out when load is high and scales in when load drops.  

**Resource requests and limits**  
- **Requests**: minimum CPU/memory a container asks for; used by the scheduler to place Pods on nodes.  
- **Limits**: maximum CPU/memory a container is allowed to use at runtime; prevents a single container from hogging resources.  
- Typical pattern: small but realistic requests, slightly higher limits, then tune based on metrics.  

***

## NGINX Ingress – Setup & Notes

High-level flow (AKS or local cluster):

1. Install NGINX Ingress controller (usually via Helm or official manifests).  
2. Expose it with a Service (often `LoadBalancer` on AKS).  
3. Create an Ingress resource that routes traffic (e.g. `/` or `/api`) to your API Service.  

Example Ingress for your FastAPI Service (assuming `fastapi-aks-service` on port 80):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fastapi-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: fastapi.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fastapi-aks-service
            port:
              number: 80
```

Notes:
- `kubernetes.io/ingress.class: nginx` ties the Ingress to the NGINX controller.  
- On AKS, the NGINX controller's Service will typically be a `LoadBalancer` with a public IP; you point DNS to that IP.  

***

## Resource Limits in Your Deployment

Update your Deployment to include resource requests and limits so HPA has something meaningful to work with:

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
        image: myacr12345.azurecr.io/fastapi-demo:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

Notes:
- CPU unit `m` = millicores (100m = 0.1 CPU).  
- HPA uses CPU utilization relative to the **request** (e.g. 50% of 100m = 50m).  

***

## Configure HPA (CPU-based)

Create an HPA targeting your Deployment:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastapi-aks-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Notes:
- `averageUtilization: 50` means "try to keep average CPU at 50% of the request per Pod."  
- Ensure the metrics server is installed in the cluster so HPA can read CPU metrics.  

Alternatively, via `kubectl`:

```bash
kubectl autoscale deployment fastapi-aks-deployment \
  --cpu-percent=50 \
  --min=2 \
  --max=10
```

***

## Build: Wire It Together

1. **Ensure Deployment has requests/limits**  
   - Update `deployment.yaml` as shown above.  

2. **Install NGINX Ingress controller**  
   - On AKS, use the official ingress-nginx Helm chart or manifest.  

3. **Create Ingress for your API**  
   - Apply `fastapi-ingress` pointing to `fastapi-aks-service`.  

4. **Create HPA**  
   - Apply `fastapi-hpa` YAML, or use `kubectl autoscale`.  

***

## Deliverable: API Auto-scales Under Load

To demonstrate:

1. Check current replicas:
   ```bash
   kubectl get deploy fastapi-aks-deployment
   kubectl get hpa fastapi-hpa
   ```

2. Generate load:
   ```bash
   # Simple loop (demo only)
   while true; do
     curl -s http://<INGRESS-IP>/ > /dev/null
   done
   ```

3. Watch scaling:
   ```bash
   kubectl get hpa fastapi-hpa -w
   kubectl get deploy fastapi-aks-deployment -w
   ```

4. Expected behavior:
   - CPU > 50% → replicas increase (up to max).  
   - CPU drops → replicas decrease (not below min).  

When you see replicas scaling up/down while API stays reachable via Ingress, Day 4 is complete.

