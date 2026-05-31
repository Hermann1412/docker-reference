# Kubernetes NodePort Service Demo

This demo shows how a **NodePort** service exposes pods to traffic from **outside** the cluster by opening a static port on the node itself.

---

## Files

| File | Kind | Purpose |
|------|------|---------|
| `deploy-app.yaml` | Deployment | Runs 2 nginx replicas labelled `app=nginx, env=prod` |
| `nodeport.yaml` | Service (NodePort) | Exposes nginx externally on node port **32410** |

---

## What Each File Does

### `nodeport.yaml` — NodePort Service
```yaml
kind: Service
spec:
  type: NodePort
  selector:
    app: nginx
    env: prod
  ports:
  - nodePort: 32410   # external port opened on the node
    port: 80          # internal cluster port
    targetPort: 80    # port nginx listens on inside the container
```
- `type: NodePort` is what makes this different from ClusterIP — Kubernetes opens port **32410** on the node itself, making the service reachable from your host machine.
- Three ports are in play:
  - **nodePort 32410** — what you hit from outside (e.g. `localhost:32410` or `<node-ip>:32410`)
  - **port 80** — the ClusterIP port used for internal cluster traffic
  - **targetPort 80** — the port nginx actually listens on inside the container
- The selector `app=nginx, env=prod` routes traffic to matching pods.

### `deploy-app.yaml` — Deployment
```yaml
kind: Deployment
spec:
  replicas: 2
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: nginx
      env: prod
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```
- Runs **2 replicas** of `nginx:alpine`, each listening on port 80.
- Labels `app=nginx, env=prod` match the NodePort service selector, so the service automatically discovers both pods.
- `revisionHistoryLimit: 3` retains only the last 3 ReplicaSets for rollback purposes.

---

## What You Did Step by Step

### 1. Applied the Deployment first
```powershell
kubectl apply -f deploy-app.yaml   # created deployment.apps/deploy-nginx
```
Kubernetes created a ReplicaSet and started spinning up 2 nginx pods.

### 2. Applied the NodePort Service
```powershell
kubectl apply -f nodeport.yaml     # created service/svc-example
```
Kubernetes opened port **32410** on the node and began routing traffic to the nginx pods.

### 3. Verified pods were running
```powershell
kubectl get pods -o wide
```
Output confirmed both pods in **Running** state on `docker-desktop`:

| Pod | IP |
|-----|----|
| `deploy-nginx-6db5c8df9c-h6gv2` | 10.1.0.111 |
| `deploy-nginx-6db5c8df9c-lqg4w` | 10.1.0.112 |

At this point you could have hit the service from your browser or terminal at `http://localhost:32410` and gotten the nginx welcome page — without needing to exec into any pod, because NodePort is externally accessible.

### 4. Cleaned up all resources
```powershell
kubectl delete -f deploy-app.yaml   # deleted deployment + its pods
kubectl delete -f nodeport.yaml     # deleted service + closed node port 32410
```

---

## Key Concepts Demonstrated

| Concept | What happened |
|---------|---------------|
| **NodePort** | Opens a static port (32410) on the node, reachable from outside the cluster |
| **Three-port mapping** | `nodePort 32410` → `port 80` (ClusterIP) → `targetPort 80` (container) |
| **External access** | Unlike ClusterIP, no need to exec into a pod — you can curl/browser from your host |
| **Label selectors** | Service found both nginx pods via `app=nginx, env=prod` labels |
| **Load balancing** | Requests on port 32410 are distributed across both running replicas |

---

## NodePort vs ClusterIP

| Feature | ClusterIP | NodePort |
|---------|-----------|----------|
| Reachable from inside cluster | Yes | Yes |
| Reachable from host machine | No | Yes (via node port) |
| Reachable from the internet | No | Yes (if firewall allows) |
| Port range | Any | 30000–32767 |
| Use case | Internal microservice comms | Dev/testing external access |
