# Kubernetes Deployment Guide for Trading System

This guide provides step-by-step instructions for deploying the Trading System architecture to a Kubernetes cluster based on the trading-system-k8s.decorator.json specification.

## Prerequisites

- Kubernetes cluster (v1.24+)
- kubectl configured with cluster access
- Sufficient cluster resources for all components
- Storage classes: `fast-ssd` and `standard`
- Optional: Ingress controller, cert-manager, Prometheus operator

---

## Step 1: Prerequisites Setup

Verify cluster connectivity and create the namespace:

```bash
# Ensure kubectl is configured
kubectl cluster-info

# Create namespace
kubectl create namespace trading-system

# Set context
kubectl config set-context --current --namespace=trading-system
```

---

## Step 2: Deploy Persistent Storage (Databases First)

Create PersistentVolumeClaims for the databases:

```bash
# Create PersistentVolumeClaims
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: trade-data-store-pvc
  namespace: trading-system
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: user-directory-pvc
  namespace: trading-system
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
EOF
```

---

## Step 3: Deploy Databases

Deploy PostgreSQL instances for trade data and user directory:

```bash
# Deploy trade-data-store
kubectl create deployment trade-data-store \
  --image=postgres:15 \
  --replicas=1 \
  --namespace=trading-system

kubectl set resources deployment trade-data-store \
  --requests=cpu=2000m,memory=4Gi \
  --limits=cpu=4000m,memory=8Gi

kubectl expose deployment trade-data-store \
  --port=5432 --type=ClusterIP

# Deploy user-directory
kubectl create deployment user-directory \
  --image=postgres:15 \
  --replicas=1 \
  --namespace=trading-system

kubectl expose deployment user-directory \
  --port=5432 --type=ClusterIP
```

---

## Step 4: Deploy Backend Services

Deploy all backend microservices:

```bash
# Deploy each service with its configuration
for service in account-service trade-service position-service \
               trade-processor security-master people-service trade-feed
do
  kubectl create deployment $service \
    --image=trading/$service:v1.0.0 \
    --namespace=trading-system
  
  # Get port from decorator data for each service
  kubectl expose deployment $service --type=ClusterIP
done

# Scale replicas per decorator spec
kubectl scale deployment account-service --replicas=3
kubectl scale deployment trade-service --replicas=5
kubectl scale deployment position-service --replicas=3
kubectl scale deployment trade-processor --replicas=2
kubectl scale deployment security-master --replicas=2
kubectl scale deployment people-service --replicas=2
kubectl scale deployment trade-feed --replicas=3
```

---

## Step 5: Deploy Frontend

Deploy the web GUI with LoadBalancer service:

```bash
kubectl create deployment web-gui \
  --image=trading/web-gui:v1.0.0 \
  --replicas=2 \
  --namespace=trading-system

kubectl expose deployment web-gui \
  --port=80 --type=LoadBalancer
```

---

## Step 6: Apply Network Policies

Configure network policies for security:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: trading-system-network-policy
  namespace: trading-system
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: trading-system
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: trading-system
EOF
```

---

## Step 7: Configure Ingress (Optional)

Set up Ingress with TLS for external access:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: trading-ingress
  namespace: trading-system
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - trading.company.com
    secretName: trading-tls
  rules:
  - host: trading.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-gui
            port:
              number: 80
EOF
```

---

## Step 8: Verify Deployment

Check that all components are running correctly:

```bash
# Check all pods are running
kubectl get pods -n trading-system

# Check services
kubectl get svc -n trading-system

# Check ingress
kubectl get ingress -n trading-system

# Tail logs for a specific service
kubectl logs -f deployment/trade-service -n trading-system

# Check pod status details
kubectl describe pods -n trading-system
```

---

## Step 9: Enable Monitoring

Deploy Prometheus ServiceMonitor for metrics collection:

```bash
# Apply Prometheus ServiceMonitor
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: trading-system-metrics
  namespace: trading-system
spec:
  selector:
    matchLabels:
      app: trading-system
  endpoints:
  - port: metrics
    interval: 30s
EOF
```

---

## Resource Requirements Summary

Based on the decorator specification:

| Component | Replicas | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|----------|-------------|----------------|-----------|--------------|
| account-service | 3 | 500m | 1Gi | 1000m | 2Gi |
| trade-service | 5 | 1000m | 2Gi | 2000m | 4Gi |
| position-service | 3 | 500m | 1Gi | 1000m | 2Gi |
| trade-processor | 2 | 1000m | 2Gi | 2000m | 4Gi |
| security-master | 2 | 250m | 512Mi | 500m | 1Gi |
| people-service | 2 | 250m | 512Mi | 500m | 1Gi |
| trade-feed | 3 | 500m | 1Gi | 1000m | 2Gi |
| trade-data-store | 1 | 2000m | 4Gi | 4000m | 8Gi |
| user-directory | 1 | 500m | 1Gi | 1000m | 2Gi |
| web-gui | 2 | 250m | 512Mi | 500m | 1Gi |

**Total Cluster Requirements:**
- CPU Requests: ~12 cores
- Memory Requests: ~24 Gi
- Storage: 110 Gi

---

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod <pod-name> -n trading-system
kubectl logs <pod-name> -n trading-system
```

### Service connectivity issues
```bash
# Test service DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup trade-service

# Test service connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://trade-service:18092/health
```

### Storage issues
```bash
kubectl get pvc -n trading-system
kubectl describe pvc trade-data-store-pvc -n trading-system
```

---

## Cleanup

To remove the entire deployment:

```bash
# Delete all resources in the namespace
kubectl delete namespace trading-system

# Delete PVCs (if not automatically deleted)
kubectl delete pvc --all -n trading-system
```

---

## Next Steps

- Configure horizontal pod autoscaling (HPA)
- Set up backup strategies for databases
- Configure alerting rules in Prometheus
- Implement CI/CD pipelines for automated deployments
- Review and tune resource limits based on actual usage

---

## References

- Architecture: trading-system.architecture.json
- Deployment Decorator: trading-system-k8s.decorator.json
- CALM Documentation: https://calm.finos.org/