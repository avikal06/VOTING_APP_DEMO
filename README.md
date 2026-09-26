# Voting App on Kubernetes (minikube)

A Kubernetes deployment of Docker's example voting app. Users vote on one page and see live results on another.

## Architecture

```
 vote (Python)  ──►  redis  ──►  worker (.NET)  ──►  db (Postgres)  ──►  result (Node.js)
   NodePort 31000                                                         NodePort 31001
```

| Component | Image | Service | Port |
|---|---|---|---|
| vote | `docker/example-voting-app-vote` | NodePort | 8080 → 80 (node 31000) |
| result | `docker/example-voting-app-result` | NodePort | 8081 → 80 (node 31001) |
| redis | `redis:alpine` | ClusterIP | 6379 |
| db | `postgres:9.4` | ClusterIP | 5432 |
| worker | `docker/example-voting-app-worker` | none (no incoming traffic) | none |

## Project layout

```
VOTING_APP_DEMO/
├── Pod/            # Bare Pod manifests + Services (first iteration)
└── deployments/    # Deployment manifests + Services (recommended)
```

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

## 1. Start the cluster

```bash
minikube start
```

```bash
minikube status
```

```bash
kubectl get nodes
```

## 2. Deploy the app (Deployments, recommended)

```bash
cd deployments
```

```bash
kubectl apply -f .
```

Wait until every deployment has rolled out:

```bash
kubectl rollout status deploy/db deploy/redis deploy/vote deploy/result deploy/worker
```

Check that everything is running:

```bash
kubectl get pods,deploy,svc -o wide
```

Expected result: vote and result have 3 replicas each, and db, redis and worker have 1 each.

### Alternative: deploy as bare Pods

Only use this if you're not using the Deployments above. Don't run both at the same time: the Services select pods by label and would send traffic to both sets.

```bash
cd Pod
```

```bash
kubectl apply -f .
```

## 3. Access the app locally

With minikube's **Docker driver on macOS**, the node IP (for example `192.168.49.2`) **cannot be reached from your Mac**, so `http://<minikube-ip>:31000` will time out. Use a tunnel instead.

### Option A: port-forward (fixed ports)

```bash
kubectl port-forward svc/vote 8080:8080 & kubectl port-forward svc/result 8081:8081 &
```

| Page | URL |
|---|---|
| Vote | http://127.0.0.1:8080 |
| Result | http://127.0.0.1:8081 |

Stop the tunnels:

```bash
pkill -f "kubectl port-forward"
```

### Option B: minikube service (random port)

Run each command in its own terminal and keep it open. It prints a `http://127.0.0.1:<port>` URL to open.

```bash
minikube service vote --url
```

```bash
minikube service result --url
```

On Linux, or with a VM driver, you can use the NodePort directly: `http://$(minikube ip):31000` for vote and `:31001` for result.

## 4. Verify

Cast a vote on the vote page. The count should update on the result page.

```bash
kubectl logs deploy/worker --tail=20
```

```bash
kubectl logs deploy/result --tail=20
```

```bash
kubectl logs deploy/db --tail=20
```

The worker should log `Connected to db` and `Connected to redis`.

## Troubleshooting

### `result` / `worker` stuck on "Waiting for db"

`kubectl logs deploy/db` shows `password authentication failed for user "postgres"`.

These older example images connect to Postgres **without a password**, but setting `POSTGRES_PASSWORD` makes Postgres require one. The db manifests set this so that password-less connections are allowed:

```yaml
- name: POSTGRES_HOST_AUTH_METHOD
  value: "trust"
```

After changing the db manifest, restart the dependent workloads:

```bash
kubectl apply -f db-deployment.yml
```

```bash
kubectl rollout restart deploy/db deploy/worker deploy/result
```

> ⚠️ `trust` lets anyone who can reach Postgres log in without a password. That's fine for a local demo, but don't use it in production.

### `relation "votes" does not exist`

This is normal right after startup: the result app queries before the worker has created the table. It goes away once the worker connects and the first vote is cast.

### `bind: address already in use` on port-forward

A port-forward is already running on that port. Either use the existing one, or stop it and start again:

```bash
pkill -f "kubectl port-forward"
```

### Check that a Service has endpoints

If a Service has no endpoints, its selector doesn't match any pod labels:

```bash
kubectl get endpointslices
```

```bash
kubectl describe svc vote
```

### Test from inside the cluster

```bash
kubectl run curltest --rm -i --restart=Never --image=curlimages/curl -- curl -s -o /dev/null -w "%{http_code}\n" http://vote:8080
```

### Validate manifests without applying

```bash
kubectl apply --dry-run=server -f .
```

## Scaling

```bash
kubectl scale deploy/vote --replicas=5
```

```bash
kubectl get pods -l app=vote
```

## Cleanup

Delete the app:

```bash
kubectl delete -f deployments/
```

Stop the cluster:

```bash
minikube stop
```

Delete the cluster entirely:

```bash
minikube delete
```
