# Kubernetes ClusterIP Service Demo

This demo shows how a **ClusterIP** service enables internal pod-to-pod communication inside a Kubernetes cluster.

---

## Files

| File | Kind | Purpose |
|------|------|---------|
| `clusterip.yaml` | Service (ClusterIP) | Exposes nginx pods internally on port 8080 |
| `deploy-app.yaml` | Deployment | Runs 3 nginx replicas labelled `app=app-example, env=prod` |
| `pod.yaml` | Pod | A busybox utility pod used as a client to test the service |

---

## What Each File Does

### `clusterip.yaml` — ClusterIP Service
```yaml
kind: Service
spec:
  ports:
  - port: 8080       # port clients inside the cluster use
    targetPort: 80   # port nginx actually listens on inside the container
  selector:
    app: app-example
    env: prod
```
- Creates a stable internal DNS name **`svc-example`** reachable by any pod in the cluster.
- Routes traffic on **port 8080** to **port 80** on any pod matching the label selector `app=app-example, env=prod`.
- No external access — ClusterIP is cluster-internal only.

### `deploy-app.yaml` — Deployment
```yaml
kind: Deployment
spec:
  replicas: 3
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: app-example
      env: prod
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```
- Runs **3 replicas** of `nginx:alpine`, each listening on port 80.
- The labels `app=app-example, env=prod` match the service selector above, so the service automatically discovers and load-balances across all 3 pods.
- `revisionHistoryLimit: 3` keeps only the last 3 ReplicaSets for rollback.

### `pod.yaml` — Client Pod (mybox)
```yaml
kind: Pod
metadata:
  name: mybox
spec:
  containers:
  - name: mybox
    image: busybox
    command: ["sleep", "3600"]
    resources:
      requests: { cpu: 100m, memory: 128Mi }
      limits:   { cpu: 250m, memory: 256Mi }
```
- A lightweight **busybox** pod that just sleeps for 1 hour — giving you a shell to test from.
- Resource requests/limits are set so the scheduler can place it efficiently without over-consuming.

---

## What You Did Step by Step

### 1. Applied all resources
```powershell
kubectl apply -f clusterip.yaml   # created service/svc-example
kubectl apply -f deploy-app.yaml  # created deployment.apps/deploy-example
kubectl apply -f pod.yaml         # created pod/mybox
```

### 2. Verified all pods were running
```powershell
kubectl get pods -o wide
```
Output confirmed 4 pods all in **Running** state on the `docker-desktop` node:
- `deploy-example-586dc685c-bfkwl` — 10.1.0.109
- `deploy-example-586dc685c-bzjmp` — 10.1.0.107
- `deploy-example-586dc685c-mzz8m` — 10.1.0.108
- `mybox`                           — 10.1.0.110

Each pod got its own cluster-internal IP, but you didn't need to know them — the service provides a single stable hostname.

### 3. Shelled into the client pod and tested the service
```sh
kubectl exec mybox -it -- /bin/sh
wget -qO- http://svc-example:8080
```
- `kubectl exec` opened an interactive shell inside `mybox`.
- `wget -qO-` sent an HTTP GET to the service by **DNS name** (`svc-example`) on port 8080.
- Kubernetes DNS automatically resolved `svc-example` to the ClusterIP of the service.
- The service forwarded the request to one of the 3 nginx pods (port 80).
- The nginx welcome page HTML was returned — confirming the service works correctly.

### 4. Cleaned up all resources
```powershell
kubectl delete -f clusterip.yaml   # deleted service
kubectl delete -f deploy-app.yaml  # deleted deployment (and its ReplicaSet + pods)
kubectl delete -f pod.yaml         # deleted mybox pod
```

---

## Key Concepts Demonstrated

| Concept | What happened |
|---------|---------------|
| **ClusterIP** | Default service type — internal only, not reachable from outside the cluster |
| **Label selectors** | The service found its backend pods using `app=app-example, env=prod` labels |
| **DNS-based discovery** | `mybox` reached nginx via `http://svc-example:8080` — no IPs hardcoded |
| **Load balancing** | With 3 replicas behind the service, requests are distributed across pods |
| **Port mapping** | Clients use 8080 (service port); nginx listens on 80 (targetPort) |
| **Utility pod pattern** | `mybox` (busybox + sleep) is a common pattern for debugging inside a cluster |
