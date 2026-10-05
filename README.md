# IMS — Kubernetes Migration

A personal project I built (an Incident Management System) and then migrated from Docker Compose to Kubernetes. It runs five services: PostgreSQL, MongoDB, Redis, a FastAPI backend and a React/nginx frontend.

Original application: https://github.com/kernal05/ims-zeotap

## What this demonstrates

- Multi-service app migrated from `docker-compose.yml` to Kubernetes manifests
- Namespace isolation (`ims-app`)
- Secrets for DB credentials, matching the app's environment variable conventions
- **Persistence:** PostgreSQL and MongoDB use PersistentVolumeClaims with the `Recreate` strategy. Verified by writing a marker row, deleting the Postgres pod, and reading the row back from the replacement pod.
- **Health checks and limits:** readiness/liveness probes and resource requests/limits on every workload
- **NetworkPolicies:** default-deny ingress, with explicit allows for frontend -> backend -> databases
- **Local images without a registry:** images are built with Docker and loaded into minikube with `minikube image load`, then used with `imagePullPolicy: Never`
- **Real debugging:** fixed a broken nginx upstream (Docker Compose container-name discovery doesn't carry over to Kubernetes' DNS-1035 service naming, which disallows underscores) by overriding the baked-in nginx config through a ConfigMap volume mount with `subPath`, with no image rebuild

## Known limitations (stated honestly)

- **NetworkPolicies are written but not enforced on minikube's default network plugin.** I tested this: a pod with no allow rule could still reach Postgres on 5432. Enforcement needs a policy-capable CNI such as Calico or Cilium.
- **Database probes are TCP, not exec.** `pg_isready` and `redis-cli ping` exec probes timed out on my local Docker-driver cluster, so I used TCP probes. On a normal cluster, exec probes are the better choice.
- This runs on a single-node local minikube cluster. It is a learning and demo setup, not a production deployment.

## Prerequisites

- minikube (or any single-node cluster) and kubectl
- The images `ims-zeotap-backend:latest` and `ims-zeotap-frontend:latest` built locally

## How to run

```bash
# 1. Start the cluster and load the locally built images
minikube start
minikube image load ims-zeotap-backend:latest
minikube image load ims-zeotap-frontend:latest

# 2. Namespace
kubectl apply -f namespace.yaml

# 3. Create the Secret (see secret-setup.md; keys use ims_user / ims_db / ims_raw)

# 4. Databases first, wait until they are Ready and PVCs are Bound
kubectl apply -f postgres.yaml -f mongodb.yaml -f redis.yaml
kubectl get pods,pvc -n ims-app

# 5. Backend and frontend
kubectl apply -f backend.yaml -f frontend-nginx-configmap.yaml -f frontend.yaml

# 6. Optional: NetworkPolicies (need Calico/Cilium to be enforced)
kubectl apply -f networkpolicy.yaml

# 7. Access the app (use any free local port)
kubectl port-forward -n ims-app svc/frontend 18080:80
curl http://localhost:18080/api/incidents
```

## Verify persistence

```bash
kubectl exec -n ims-app deploy/postgres -- sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "create table if not exists persist_test(v text); insert into persist_test values ('"'"'survived'"'"');"'
kubectl delete pod -n ims-app -l app=postgres
kubectl rollout status deployment/postgres -n ims-app --timeout=300s
kubectl exec -n ims-app deploy/postgres -- sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "select * from persist_test;"'
```

If the `survived` row is returned from the new pod, the volume is working.
