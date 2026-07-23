## Secret setup (not committed with real values)

\`\`\`bash
kubectl create namespace ims-app

kubectl create secret generic ims-secret \
  --namespace=ims-app \
  --from-literal=POSTGRES_USER=<value> \
  --from-literal=POSTGRES_PASSWORD=<value> \
  --from-literal=POSTGRES_DB=<value> \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=<value> \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD=<value> \
  --from-literal=MONGO_INITDB_DATABASE=<value> \
  --from-literal=POSTGRES_URL=<value> \
  --from-literal=MONGODB_URL=<value> \
  --from-literal=REDIS_URL=<value>
\`\`\`
