# TaskBoard Kubernetes manifests

Numbered manifests deploy the full stack: Postgres, API, frontend, and Ingress.

## Architecture

```
Browser → Ingress (taskboard.local)
            ├─ /api  → backend Service :4000 → API pod
            └─ /     → frontend Service :80  → nginx + SPA
                              backend pod → postgres Service :5432 → Postgres pod + PVC
```

## Apply order

Files are numbered so you can read and apply them in dependency order:

| File | Kind | Purpose |
|------|------|---------|
| `00-namespace.yaml` | Namespace | Isolates all TaskBoard resources in `production` |
| `01-secrets.yaml` | Secret | DB credentials, `DATABASE_URL`, `JWT_SECRET` |
| `02-configmap.yaml` | ConfigMap | Non-secret backend config (`NODE_ENV`, `CORS_ORIGIN`, etc.) |
| `03-postgres-pvc.yaml` | PersistentVolumeClaim | Requests 1Gi disk for Postgres data |
| `04-postgres-deployment.yaml` | Deployment | Runs one Postgres pod with probes and volume mount |
| `05-postgres-service.yaml` | Service | Stable DNS `postgres:5432` inside the cluster |
| `06-backend-deployment.yaml` | Deployment | API pod; init container waits for Postgres |
| `07-backend-service.yaml` | Service | Exposes API on port 4000 inside the cluster |
| `08-frontend-deployment.yaml` | Deployment | nginx serving the built React SPA |
| `09-frontend-service.yaml` | Service | Exposes frontend on port 80 inside the cluster |
| `10-ingress.yaml` | Ingress | Routes external HTTP to frontend and backend |

### What each resource type does

- **Namespace** — logical boundary; `kubectl get all -n production` shows only this app.
- **Secret** — sensitive values (passwords, tokens). Injected as env vars into pods.
- **ConfigMap** — non-sensitive config. Changing it does not restart pods until you roll the Deployment.
- **PVC** — asks the cluster for persistent disk. Postgres data survives pod restarts.
- **Deployment** — desired state for pods (image, env, probes). ReplicaSet is created automatically.
- **Service** — stable cluster IP + DNS for a set of pods. `ClusterIP` = internal only.
- **Ingress** — maps host/path from outside the cluster to Services.

## Prerequisites

- `kubectl` configured for your cluster
- Docker (for building images)

## OrbStack (local, single node)

1. Enable Kubernetes in OrbStack and verify:

   ```bash
   kubectl cluster-info
   ```

2. Build images (OrbStack shares the Docker daemon with Kubernetes):

   ```bash
   docker build -t taskboard-api:local ./taskboard-api
   docker build -t taskboard-front:local --build-arg VITE_API_URL=/api ./taskboard-front
   ```

3. Copy secrets if needed (dev values are already in `01-secrets.yaml`):

   ```bash
   cp k8s/secrets.example.yaml k8s/01-secrets.yaml
   # edit passwords and JWT_SECRET
   ```

4. Apply manifests:

   ```bash
   kubectl apply -f k8s/
   ```

5. Watch rollout:

   ```bash
   kubectl get pods -n production -w
   ```

6. Open the app using the OrbStack service address (e.g. `http://192.168.x.x` from `kubectl get svc -n production frontend`). Frontend nginx proxies `/api` to the backend automatically.

   Optional: add `127.0.0.1 taskboard.local` to `/etc/hosts` if you use Ingress with that hostname.

### Debug commands

```bash
kubectl get all -n production
kubectl describe pod -n production -l app=backend
kubectl logs -f deployment/backend -n production
kubectl logs -f deployment/postgres -n production
kubectl get ingress -n production
```

## k3s (multi-node cluster)

Use the same YAML. Differences:

1. **Images** — build locally, push to a registry (e.g. GHCR), then update image names in `06-backend-deployment.yaml` and `08-frontend-deployment.yaml`:

   ```bash
   docker build -t ghcr.io/<your-user>/taskboard-api:latest ./taskboard-api
   docker build -t ghcr.io/<your-user>/taskboard-front:latest --build-arg VITE_API_URL=/api ./taskboard-front
   docker push ghcr.io/<your-user>/taskboard-api:latest
   docker push ghcr.io/<your-user>/taskboard-front:latest
   ```

   Set `imagePullPolicy: Always` and add `imagePullSecrets` if the registry is private.

2. **Ingress host** — change `taskboard.local` in `10-ingress.yaml` and `CORS_ORIGIN` in `02-configmap.yaml` to your domain or node IP. Add a matching `/etc/hosts` entry pointing at any node IP.

3. **Storage** — k3s provides `local-path` StorageClass by default. Verify with `kubectl get sc`.

4. **Workers** — no extra config needed; the scheduler places pods on workers. Postgres PVC binds to the node where its pod runs.

## Secrets

- `secrets.example.yaml` — template with placeholders (safe to commit).
- `01-secrets.yaml` — dev defaults for local learning. For real deployments, use strong passwords and avoid committing production secrets.

Required secret keys:

| Key | Used by |
|-----|---------|
| `POSTGRES_USER` | Postgres Deployment |
| `POSTGRES_PASSWORD` | Postgres Deployment |
| `POSTGRES_DB` | Postgres Deployment |
| `DATABASE_URL` | Backend Deployment |
| `JWT_SECRET` | Backend Deployment (min 16 characters) |

## Common issues

| Symptom | Likely cause |
|---------|----------------|
| `db-deployment` / invalid Deployment | Old stub file; use `04-postgres-deployment.yaml` |
| Backend exits immediately | Missing or wrong `DATABASE_URL` / `JWT_SECRET` |
| Frontend loads but register/login fails (405) | You hit the frontend Service directly; `/api` must be proxied to backend (see `nginx.conf`) or accessed via Ingress |
| Frontend calls wrong API | Rebuild frontend image after changing `VITE_API_URL` (build-time, not runtime) |
| `ImagePullBackOff` on k3s | Push images to a registry or load them on each node |
| Postgres empty after delete | PVC was deleted; data lives on the PVC, not the Deployment |

## Tear down

```bash
kubectl delete -f k8s/
```

To also remove persisted database data:

```bash
kubectl delete pvc postgres-data -n production
```
