# TaskBoard Kubernetes

Manifests for deploying TaskBoard (Postgres, API, frontend, Ingress) into a `taskboard` namespace.

Related repos: [taskboard-api](https://github.com/FinnTheHero/taskboard-api) · [taskboard-front](https://github.com/FinnTheHero/taskboard-front)

App images are public on GHCR (no pull secret required):

- `ghcr.io/finnthehero/taskboard-api:latest`
- `ghcr.io/finnthehero/taskboard-front:latest`

## Architecture

```mermaid
flowchart LR
  Browser([Browser])

  subgraph cluster["Namespace: taskboard"]
    Ingress["Ingress<br/>taskboard.local<br/>/ and /api"]

    FrontendSvc["Service<br/>frontend :80"]
    BackendSvc["Service<br/>backend :4000"]
    PostgresSvc["Service<br/>postgres :5432"]

    FrontendPod["Deployment<br/>frontend<br/>nginx + SPA"]
    BackendPod["Deployment<br/>backend<br/>API"]
    PostgresPod["Deployment<br/>postgres<br/>postgres:16"]

    PVC[("PVC<br/>postgres-data<br/>1Gi")]
    Secret[("Secret<br/>taskboard-secrets")]
    ConfigMap[("ConfigMap<br/>backend-config")]
  end

  Browser --> Ingress
  Ingress -->|"/"| FrontendSvc
  Ingress -->|"/api"| BackendSvc
  FrontendSvc --> FrontendPod
  BackendSvc --> BackendPod
  BackendPod --> PostgresSvc
  PostgresSvc --> PostgresPod
  PostgresPod --> PVC
  Secret -.-> BackendPod
  Secret -.-> PostgresPod
  ConfigMap -.-> BackendPod
```

**Request flow:** Browser → Ingress → frontend (`/`) or backend (`/api`) → Postgres.

| File | Kind | Purpose |
|------|------|---------|
| `00-namespace.yaml` | Namespace | Isolates resources in `taskboard` |
| `01-secrets.yaml` | Secret | DB credentials, `DATABASE_URL`, `JWT_SECRET` |
| `02-configmap.yaml` | ConfigMap | Non-secret backend config |
| `03-postgres-pvc.yaml` | PersistentVolumeClaim | 1Gi disk for Postgres data |
| `04-postgres-deployment.yaml` | Deployment | Postgres pod with probes + volume |
| `05-postgres-service.yaml` | Service | Cluster DNS `postgres:5432` |
| `06-backend-deployment.yaml` | Deployment | API; init container waits for DB |
| `07-backend-service.yaml` | Service | Exposes API on port 4000 |
| `08-frontend-deployment.yaml` | Deployment | nginx serving the React SPA |
| `09-frontend-service.yaml` | Service | Exposes frontend on port 80 |
| `10-ingress.yaml` | Ingress | Routes external HTTP to services |

## Prerequisites

- `kubectl` configured for your cluster
- An Ingress controller (e.g. ingress-nginx) if you use [`10-ingress.yaml`](10-ingress.yaml)
- Docker (only when rebuilding and pushing images)

## Deploy

1. Verify the cluster:

   ```bash
   kubectl cluster-info
   ```

2. Copy secrets if needed (dev values are already in `01-secrets.yaml`):

   ```bash
   cp secrets.example.yaml 01-secrets.yaml
   # edit passwords and JWT_SECRET
   ```

3. Apply manifests (Deployments pull from GHCR):

   ```bash
   kubectl apply -f .
   ```

4. Watch rollout:

   ```bash
   kubectl get pods -n taskboard -w
   ```

5. Open via Ingress IP (`kubectl get ingress -n taskboard`) or add `127.0.0.1 taskboard.local` to `/etc/hosts`.

After pushing new GHCR builds, restart so nodes re-pull `:latest` (`imagePullPolicy: Always`):

```bash
kubectl rollout restart deployment/backend deployment/frontend -n taskboard
```

### Rebuild and push images

When you change the apps, rebuild, retag, and push:

```bash
docker build -t taskboard-api:local ../taskboard-api
docker build -t taskboard-front:local --build-arg VITE_API_URL=/api ../taskboard-front

docker tag taskboard-api:local ghcr.io/finnthehero/taskboard-api:latest
docker tag taskboard-front:local ghcr.io/finnthehero/taskboard-front:latest

docker push ghcr.io/finnthehero/taskboard-api:latest
docker push ghcr.io/finnthehero/taskboard-front:latest
```

Packages are public — no `imagePullSecrets` on the Deployments.

### Debug

```bash
kubectl get all -n taskboard
kubectl describe pod -n taskboard -l app=backend
kubectl logs -f deployment/backend -n taskboard
kubectl logs -f deployment/postgres -n taskboard
kubectl get ingress -n taskboard
```

## Cluster notes

1. **Ingress** — ensure an Ingress controller is installed. Update the host in `10-ingress.yaml` and `CORS_ORIGIN` in `02-configmap.yaml` to your real hostname or load-balancer URL.

2. **Storage** — the PVC uses the cluster default StorageClass. Check what is available:

   ```bash
   kubectl get sc
   ```

   If there is no default, set `spec.storageClassName` on `03-postgres-pvc.yaml` to a class that supports `ReadWriteOnce`.

## Secrets

| Key | Used by |
|-----|---------|
| `POSTGRES_USER` | Postgres |
| `POSTGRES_PASSWORD` | Postgres |
| `POSTGRES_DB` | Postgres |
| `DATABASE_URL` | Backend |
| `JWT_SECRET` | Backend (min 16 characters) |

- `secrets.example.yaml` — template with placeholders (safe to commit)
- `01-secrets.yaml` — local dev defaults; do not commit production secrets

## Common issues

| Symptom | Likely cause |
|---------|--------------|
| Backend exits immediately | Missing or wrong `DATABASE_URL` / `JWT_SECRET` |
| Frontend loads but API fails (405) | Hit frontend Service directly; use Ingress so `/api` reaches backend |
| Frontend calls wrong API | Rebuild after changing `VITE_API_URL` (build-time), then push GHCR and restart |
| `ImagePullBackOff` | Wrong GHCR tag/name, package not public, or network/registry issue |
| Postgres empty after delete | PVC was deleted; data lives on the PVC |

## Tear down

```bash
kubectl delete -f .
kubectl delete pvc postgres-data -n taskboard   # also wipe DB data
```
