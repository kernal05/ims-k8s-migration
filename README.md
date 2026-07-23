# IMS — Kubernetes Migration

Migrated a real 5-service production app (Incident Management System) from 
Docker Compose to Kubernetes: PostgreSQL, MongoDB, Redis, FastAPI backend, 
React/nginx frontend. Original project: https://github.com/kernal05/ims-zeotap

## What this demonstrates
- Multi-service app migrated from docker-compose.yml to K8s manifests
- Namespace isolation (ims-app)
- Secrets management for DB credentials, matching real app env var conventions
- Local image usage without a registry: built with Docker, bridged into 
  containerd via `ctr -n k8s.io images import` (Kubernetes' dockershim removal 
  means kubelet only reads from containerd's image store, not Docker's)
- nodeSelector + imagePullPolicy: Never to pin pods to the node holding 
  locally-built images
- Real debugging: fixed a broken nginx upstream reference (Docker Compose 
  container-name-based service discovery doesn't carry over to Kubernetes' 
  stricter DNS-1035 naming, which disallows underscores) by overriding the 
  baked-in nginx config via a ConfigMap volume mount with subPath — no image 
  rebuild required
- Verified end-to-end request flow: frontend Service -> nginx -> backend 
  Service -> backend pod -> Postgres, confirmed via port-forward + curl

## How to run
\`\`\`bash
kubectl create namespace ims-app
# create secret — see secret-setup.md
kubectl apply -f postgres.yaml
kubectl apply -f mongodb.yaml
kubectl apply -f redis.yaml
kubectl apply -f backend.yaml
kubectl apply -f frontend-nginx-configmap.yaml
kubectl apply -f frontend.yaml
kubectl port-forward -n ims-app svc/frontend 8080:80
curl http://localhost:8080/api/incidents
\`\`\`
