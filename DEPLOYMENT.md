# Deployment Guide – Moodle on Azure Kubernetes Service (AKS)

## Prerequisites
- Docker Desktop installed
- `kubectl` installed
- Azure CLI (`az`) installed
- Access to the Azure subscription
- Access to Docker Hub account (`jiaying0811`)
- Same AKS cluster as Odoo (`odoo17-aks`)

---

## Running Locally (Docker)

### First time setup
```bash
git pull
docker compose up
```

Access the app at `http://localhost:8080`

### Updating local database (after making changes in the app)
```bash
docker exec esm-moodle-mariadb-1 mysqldump -u moodleuser -pmoodlepass moodle > moodle_dump.sql

git add moodle_dump.sql
git commit -m "Update database dump"
git push
```

### Teammates pulling latest data
```bash
git pull
docker compose down -v
docker compose up
```

---

## Kubernetes Deployment (AKS)

### Architecture
| Component | Details |
|---|---|
| Cloud Provider | Microsoft Azure (Azure for Students) |
| Kubernetes Cluster | AKS – `odoo17-aks` in resource group `odoo17-rg` |
| Node | 1x Standard_D2as_v4 (2 vCPU, 8GB RAM) — shared with Odoo |
| Docker Image | `jiaying0811/esm-moodle:latest` (multi-platform: amd64 + arm64) |
| Database | MariaDB – database name: `moodle` |
| Public URL | `http://85.211.202.72` |

### Kubernetes Files
```
k8s/
├── namespace.yaml    # creates 'moodle' namespace
├── mariadb.yaml      # MariaDB PVC + Deployment + Service
└── moodle.yaml       # Moodle PVC + Deployment + LoadBalancer Service
```

---

## First-Time AKS Deployment

### 1. Build and push multi-platform image (from Mac)
```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t jiaying0811/esm-moodle:latest --push .
```

### 2. Connect to the cluster
```bash
az login
az aks get-credentials --resource-group odoo17-rg --name odoo17-aks
```

### 3. Apply Kubernetes manifests
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/mariadb.yaml
kubectl apply -f k8s/moodle.yaml
```

### 4. Wait for pods to be ready
```bash
kubectl get pods -n moodle
```
Both `moodle` and `mariadb` pods should show `1/1 Running`.

### 5. Restore the database
Get the mariadb pod name from the previous step, then:
```bash
# Copy the dump into the mariadb pod
kubectl cp moodle_dump.sql moodle/<mariadb-pod-name>:/tmp/moodle_dump.sql

# Restore it
kubectl exec -n moodle <mariadb-pod-name> -- bash -c "mysql -u moodleuser -pmoodlepass moodle < /tmp/moodle_dump.sql"
```

### 6. Get the public IP
```bash
kubectl get svc -n moodle
```
Wait until `EXTERNAL-IP` is assigned (may take 1–2 minutes).

### 7. Update the Moodle site URL in the database
```bash
kubectl exec -n moodle <mariadb-pod-name> -- mysql -u moodleuser -pmoodlepass moodle -e \
  "UPDATE mdl_config SET value='http://<EXTERNAL-IP>' WHERE name='wwwroot';"
```

### 8. Restart Moodle
```bash
kubectl rollout restart deployment/moodle -n moodle
```

Access the app at `http://<EXTERNAL-IP>`

---

## Updating the Docker Image

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t jiaying0811/esm-moodle:latest --push .
kubectl rollout restart deployment/moodle -n moodle
```

---

## Useful Commands

```bash
# Check pod status
kubectl get pods -n moodle

# Check service and public IP
kubectl get svc -n moodle

# View Moodle logs
kubectl logs deployment/moodle -n moodle --tail=50

# View MariaDB logs
kubectl logs deployment/mariadb -n moodle --tail=50

# Restart deployments
kubectl rollout restart deployment/moodle -n moodle
kubectl rollout restart deployment/mariadb -n moodle
```
