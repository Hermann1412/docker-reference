# Kubernetes Deployment — Hands-on Session

## What is a Deployment?

A **Deployment** is a higher-level abstraction that sits on top of a ReplicaSet. It adds:

- **Rolling updates** — replace pods one by one with zero downtime when you change the image or config.
- **Rollback** — revert to a previous revision instantly.
- **Revision history** — keeps a configurable number of old ReplicaSets so you can roll back.

The Deployment controller creates and owns a ReplicaSet; you never manage the ReplicaSet directly.

---

## Object hierarchy

```
Deployment
  └── ReplicaSet  (deploy-example-7b944585d7)
        ├── Pod   (deploy-example-7b944585d7-2bvlq)
        ├── Pod   (deploy-example-7b944585d7-4tn2x)
        └── Pod   (deploy-example-7b944585d7-9dlrr)
```

The hash (`7b944585d7`) is computed from the pod template spec. A new hash — and therefore a new ReplicaSet — is created every time you change the template (e.g. a new image tag).

---

## Manifest — `deploy-example.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-example
spec:
  replicas: 3
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: nginx
      env: prod
  template:
    metadata:
      labels:
        app: nginx
        env: prod
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 80
```

| Field | Value | Meaning |
|---|---|---|
| `replicas` | 3 | Maintain 3 running pods |
| `revisionHistoryLimit` | 3 | Keep the last 3 old ReplicaSets for rollback |
| `selector.matchLabels` | `app=nginx, env=prod` | Deployment owns pods with these labels |
| `image` | `nginx:alpine` | Lightweight nginx image |
| `cpu request / limit` | 100m / 250m | Guaranteed / max CPU per pod |
| `memory request / limit` | 128Mi / 256Mi | Guaranteed / max memory per pod |
| `containerPort` | 80 | Port nginx listens on inside the container |

---

## Commands Executed

### 1. Apply the manifest
```powershell
kubectl apply -f deploy-example.yaml
# deployment.apps/deploy-example created
```
Kubernetes creates the Deployment, which immediately creates a ReplicaSet, which spawns 3 pods.

---

### 2. List pods (wide output)
```powershell
kubectl get pods -o wide
```
```
NAME                              READY   STATUS    RESTARTS   AGE   IP          NODE
deploy-example-7b944585d7-2bvlq   1/1     Running   0          73s   10.1.0.22   docker-desktop
deploy-example-7b944585d7-4tn2x   1/1     Running   0          73s   10.1.0.23   docker-desktop
deploy-example-7b944585d7-9dlrr   1/1     Running   0          73s   10.1.0.24   docker-desktop
```
- Pod names follow the pattern `<deployment>-<rs-hash>-<random-suffix>`.
- All 3 show `1/1 Running` — container ready, no restarts.
- Each pod got its own cluster IP (`10.1.0.22–24`).

---

### 3. Describe a pod (by prefix, then by full name)
```powershell
kubectl describe pod deploy-example                          # matches first pod
kubectl describe pod deploy-example-7b944585d7-2bvlq        # specific pod
```

Key details from the output:

| Field | Value |
|---|---|
| Node | `docker-desktop / 192.168.65.3` |
| Labels | `app=nginx`, `env=prod`, `pod-template-hash=7b944585d7` |
| Controlled By | `ReplicaSet/deploy-example-7b944585d7` |
| QoS Class | `Burstable` (requests < limits) |
| Image pull | `already present on machine` — no download needed |

**QoS classes explained:**

| Class | Condition |
|---|---|
| `Guaranteed` | requests == limits for every container |
| `Burstable` | requests < limits (our case) |
| `BestEffort` | no requests or limits set at all |

**Projected volume (`kube-api-access-*`):** automatically mounted in every pod — provides the service account token, the cluster CA certificate, and namespace metadata so the pod can talk to the Kubernetes API if needed.

---

### 4. Check the Deployment
```powershell
kubectl get deploy
```
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
deploy-example   3/3     3            3           3m20s
```
- **READY** `3/3` — all desired pods are healthy.
- **UP-TO-DATE** — pods match the latest template.
- **AVAILABLE** — pods are serving traffic (passed readiness checks).

---

### 5. Describe the ReplicaSet
```powershell
kubectl describe rs
```
```
Name:           deploy-example-7b944585d7
Controlled By:  Deployment/deploy-example
Replicas:       3 current / 3 desired
Annotations:
  deployment.kubernetes.io/desired-replicas: 3
  deployment.kubernetes.io/max-replicas: 4
  deployment.kubernetes.io/revision: 1
```

- `Controlled By: Deployment/deploy-example` — you never created this ReplicaSet manually; the Deployment owns it.
- `max-replicas: 4` — during a rolling update, Kubernetes is allowed to temporarily run one extra pod (`replicas + maxSurge`, default surge = 1).
- `revision: 1` — this is the first (and only) template version deployed.

---

### 6. Delete the Deployment
```powershell
kubectl delete -f deploy-example.yaml
# deployment.apps "deploy-example" deleted
```
Deleting the Deployment cascades: Deployment → ReplicaSet → all 3 pods are terminated.

---

## Deployment vs ReplicaSet — quick comparison

| Feature | ReplicaSet | Deployment |
|---|---|---|
| Maintains pod count | Yes | Yes (via owned RS) |
| Rolling updates | No | Yes |
| Rollback | No | Yes (`kubectl rollout undo`) |
| Revision history | No | Yes (`revisionHistoryLimit`) |
| Typical usage | Rarely used directly | Standard way to run stateless apps |
