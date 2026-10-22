# Kubernetes Lab

A hands-on Kubernetes portfolio project running on a local **kind** cluster. It demonstrates application deployments, service discovery, configuration, health checks, resource management, and persistent storage through two independent workloads:

- **Notes API:** a Python Flask API backed by Redis and a PersistentVolumeClaim.
- **NGINX demo:** a three-replica web deployment using a ConfigMap, a Secret, resource requests/limits, and readiness/liveness probes.

This is a learning environment, not a production deployment.

## Architecture

```text
Local machine                         kind cluster: lab
                                      namespace: development

curl / browser -- port-forward --> notes-api Service :5000
                                      |
                                      v
                                   Flask Pod (notes-api:v1)
                                      |
                                      | REDIS_HOST=redis
                                      v
                                   redis Service :6379
                                      |
                                      v
                                   Redis Pod (redis:alpine)
                                      |
                                      v
                                   /data -> redis-data PVC (1Gi, RWO)

curl / browser -- port-forward --> web Service :80
                                      |
                                      v
                                   NGINX Deployment (3 replicas)
                                      |-- nginx-html ConfigMap -> HTML files
                                      `-- db-secret -> DB_PASSWORD environment variable
```

All Services use `ClusterIP`; local access is provided through `kubectl port-forward`. The NGINX workload is independent of the Notes API and Redis. Its Secret demonstrates environment-variable injection; it does not configure a database connection.

Redis runs as one replica with the `Recreate` deployment strategy. Its PVC is mounted at `/data`, and the claim relies on the cluster's default StorageClass because no StorageClass is specified in the manifest.

## Repository layout

```text
.
├── README.md
├── configmap.yaml                  # NGINX HTML content
├── deployment.yaml                 # NGINX deployment (web)
├── service.yaml                    # NGINX Service (web)
└── notes-api/
    ├── flask-deployment.yaml        # Flask deployment (notes-api)
    ├── flask-service.yaml           # API Service, port 5000
    ├── redis-deployment.yaml        # Redis deployment and /data mount
    ├── redis-service.yaml           # Redis Service, port 6379
    └── redis-pvc.yaml               # 1Gi ReadWriteOnce claim
```

The application source and Dockerfile live in the separate [Notes API repository](https://github.com/a-elef93/notes-api). This repository contains the Kubernetes manifests.

## Prerequisites

- Docker installed and running.
- `kind`, `kubectl`, Git, and `curl` installed.
- A default StorageClass capable of provisioning the Redis PVC in the kind cluster.

Use the repositories as sibling directories:

```text
devOps-lab/
├── kubernetes-lab/
└── notes-api/
```

## Setup

### 1. Get the repositories

For a fresh checkout:

```bash
mkdir -p ~/Documents/dev/devOps-lab
cd ~/Documents/dev/devOps-lab
git clone https://github.com/a-elef93/kubernetes-lab.git
git clone https://github.com/a-elef93/notes-api.git
cd kubernetes-lab
```

If both repositories already exist, open `~/Documents/dev/devOps-lab/kubernetes-lab` and continue. All commands below run from that directory unless stated otherwise.

### 2. Prepare the cluster and namespace

```bash
kind get clusters
# Run only if a cluster named lab does not already exist:
kind create cluster --name lab

kubectl config use-context kind-lab
kubectl get nodes
kubectl create namespace development --dry-run=client -o yaml | kubectl apply -f -
kubectl get storageclass
```

### 3. Build and load the API image

```bash
docker build -t notes-api:v1 ../notes-api
kind load docker-image notes-api:v1 --name lab
```

The manifest uses `image: notes-api:v1` with `imagePullPolicy: Never`. The image is built locally and is not published to a container registry; loading it into kind is required before starting the API.

### 4. Deploy Redis and the Notes API

```bash
kubectl -n development apply -f notes-api/redis-pvc.yaml
kubectl -n development apply -f notes-api/redis-service.yaml
kubectl -n development apply -f notes-api/redis-deployment.yaml
kubectl -n development rollout status deployment/redis --timeout=120s

kubectl -n development apply -f notes-api/flask-service.yaml
kubectl -n development apply -f notes-api/flask-deployment.yaml
kubectl -n development rollout status deployment/notes-api --timeout=120s

kubectl -n development get pods,services,pvc
```

Confirm that `redis-data` is `Bound` and both deployments are running. The API reaches Redis through the Service name `redis` within the same namespace.

### 5. Access and test the API

Keep this command running in a terminal:

```bash
kubectl -n development port-forward service/notes-api 5000:5000
```

In another terminal:

```bash
curl -i -X POST http://localhost:5000/notes \
  -H 'Content-Type: application/json' \
  -d '{"text":"Hello from Kubernetes"}'

curl -s http://localhost:5000/notes
```

The POST returns HTTP `201` with `{"message":"saved"}`. GET returns a JSON object containing the stored notes, for example:

```json
{"notes":["Hello from Kubernetes"]}
```

A POST without a non-empty `text` field returns HTTP `400` with `{"error":"text is required"}`. Previous notes remain in the list when reusing the same Redis data.

## Verify Redis persistence

POST and GET requests, and data retention after Redis Pod replacement, were confirmed during the lab. To repeat a controlled persistence check after creating a note:

```bash
# Force a Redis snapshot before replacing the Pod.
kubectl -n development exec deployment/redis -- redis-cli SAVE

kubectl -n development delete pod -l app=redis
kubectl -n development rollout status deployment/redis --timeout=120s
kubectl -n development exec deployment/redis -- redis-cli PING

curl -s http://localhost:5000/notes
```

Expect `OK` from `SAVE`, `PONG` from `PING`, and the previously stored note from GET. Keep the API port-forward running during the check.

This verifies that a replacement Redis Pod can reuse persisted data on the same PVC. The manifest does not explicitly configure AOF or a snapshot schedule; forcing `SAVE` makes this check reproducible. A PVC alone does not guarantee that every in-memory write has reached disk. This test does not demonstrate recovery from node failure, cluster deletion, or loss of the underlying storage.

## Optional: deploy the NGINX demo

Create `db-secret` **only when deploying the root NGINX workload**. Neither Flask nor Redis uses this Secret. Use a disposable demonstration value, never a real credential:

```bash
kubectl -n development create secret generic db-secret \
  --from-literal=DB_PASSWORD='local-demo-only' \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n development apply -f configmap.yaml
kubectl -n development apply -f deployment.yaml
kubectl -n development apply -f service.yaml
kubectl -n development rollout status deployment/web --timeout=120s
kubectl -n development get pods -l app=web

kubectl -n development port-forward service/web 8080:80
```

In another terminal, run `curl http://localhost:8080` or open that address in a browser. The page is supplied by the `nginx-html` ConfigMap mounted at `/usr/share/nginx/html`.

The `web` deployment uses `nginx:1.28` with three replicas. Each container requests `100m` CPU and `64Mi` memory, with limits of `200m` CPU and `128Mi` memory. HTTP readiness and liveness probes check `/` on port `80`.

## Troubleshooting

Start with:

```bash
kubectl config current-context
kubectl -n development get pods,services,pvc
kubectl -n development get events --sort-by=.metadata.creationTimestamp
kubectl -n development logs deployment/notes-api
kubectl -n development logs deployment/redis
```

| Symptom | Checks and resolution |
| --- | --- |
| `ErrImageNeverPull` for the API | Build `notes-api:v1`, load it with `kind load docker-image notes-api:v1 --name lab`, and confirm the image tag matches the manifest. |
| API still runs old code after rebuilding | Rebuild and reload the image, then run `kubectl -n development rollout restart deployment/notes-api`. Reapplying an unchanged manifest does not restart Pods. |
| Redis PVC remains `Pending` | Run `kubectl -n development describe pvc redis-data` and `kubectl get storageclass`. Check that a default StorageClass and a working provisioner are available. |
| API returns an error connecting to Redis | Check Redis logs, Pod status, and the `redis` Service. Both workloads must be in `development`, and `REDIS_HOST` must match the Service name. Retry after Redis finishes starting. |
| NGINX shows `CreateContainerConfigError` | Verify `db-secret` exists in `development` and contains the `DB_PASSWORD` key. |
| NGINX volume fails to mount | Verify the `nginx-html` ConfigMap exists in `development`. |
| Local port is already in use | Use another local port, such as `kubectl -n development port-forward service/notes-api 5001:5000`, then request `http://localhost:5001/notes`. |
| Port-forward stops working | Restart it after the selected Pod is replaced, and confirm the intended deployment is running. |
| Resources appear missing | Confirm the context is `kind-lab` and include `-n development` when inspecting resources. |

For a specific failing Pod, use `kubectl -n development describe pod <pod-name>` to inspect its events.

## Scope and limitations

- **Local environment:** kind runs Kubernetes locally in Docker. This lab does not demonstrate a cloud or production deployment.
- **Local application image:** `notes-api:v1` must be rebuilt and loaded manually; there is no published application image or automated release pipeline in these manifests.
- **Single Redis instance:** one replica and the `Recreate` strategy mean Redis is temporarily unavailable during replacement. There is no replication, failover, or high availability.
- **Limited durability validation:** persistence was tested across Redis Pod replacement. There are no backup/restore procedures or guarantees for kind cluster deletion or host failure.
- **Basic application runtime:** the supplied API starts with Flask's development server. The API and Redis manifests do not define resource requests/limits or readiness/liveness probes; those features are demonstrated by NGINX only.
- **Security and access:** the manifests do not configure API authentication, Redis authentication, TLS, Ingress, or NetworkPolicies. The NGINX Secret is an injection exercise, not a production secrets-management solution.
- **Image reproducibility:** `redis:alpine` is a moving tag; images are not pinned by digest.

## Cleanup

To remove only the application workloads while keeping the Redis PVC:

```bash
kubectl -n development delete -f notes-api/flask-deployment.yaml -f notes-api/flask-service.yaml --ignore-not-found
kubectl -n development delete -f notes-api/redis-deployment.yaml -f notes-api/redis-service.yaml --ignore-not-found
kubectl -n development delete -f deployment.yaml -f service.yaml -f configmap.yaml --ignore-not-found
kubectl -n development delete secret db-secret --ignore-not-found
```

Deleting the PVC may delete its backing storage, depending on the volume's reclaim policy. Deleting the kind cluster can also remove the local storage used by this lab. Run the following only when the lab data is no longer needed:

```bash
kubectl -n development delete -f notes-api/redis-pvc.yaml
# Optional: remove the entire local lab cluster.
kind delete cluster --name lab
```
